---
layout: post
title: "Sending millions of HTTP requests using GenStage"
author: Damir Gainetdinov
date: 2019-01-28 17:28
---

Among many other features, Adjust Dashboard allows our clients to see how their Facebook campaigns perform. A customer can connect their Facebook account to the Dashboard. After a while number of clicks, impressions and spend appear under Facebook network trackers in the Dashboard. The apparent simplicity of this feature is deceiving. To make it work, we need to fetch data from Facebook for every client perpetually.

In today's blog post I would like to show how using a proper back-pressure mechanism helps us send millions of HTTP requests to Facebook per day and how we implemented it using GenStage.

<!--more-->

## But first, some context

One Adjust account can have multiple Facebook accounts associated with it. A client adds Facebook accounts using OAuth authentication through [Adjust MMP (Mobile Measurement Partner)](https://www.facebook.com/business/solutions-explorer/measurement/584261675239027/Adjust) Facebook app. Every Facebook account can have multiple so-called AdsAccounts. Clients use individual AdsAccounts to run their Facebook campaigns. The information about campaigns performance is available via [Facebook Ads Insights Marketing API](https://developers.facebook.com/docs/marketing-api/insights). Having Facebook accounts integrated with proper credentials and AdsAccounts synced, one can finally fetch data from Facebook.
We picked Elixir as the implementation language for the project responsible for getting data from Facebook.

## Original implementation

The original implementation used the easiest and the most straightforward way to run code concurrently in Elixir - `Task.async`. We would iterate through all Adjust accounts which have Facebook accounts and spawn one process per Adjust account. Then in each of these processes, we would fetch data from Facebook concurrently firing HTTP requests to Facebook for all Facebook AdsAccounts available. One request — one task. Then all tasks are sent to `Task.await`, the fetched data is put into a queue and a `Processor` process is started per every Adjust `account_id`. Each `Processor` process gets the data from the queue, does some additional transformations and stores the data to the database.

As you can see, the original implementation was pretty straightforward: get all AdsAccounts, fetch the data using `Tasks.async`/`Task.await`, put the fetched data into the queue and process it.

However, over time, we started to observe the limitations of this architecture. We got more and more clients with Facebook accounts integrated, meaning we would spawn more and more concurrent processes to fetch Facebook data. Not only Facebook API was not happy about getting so many of these requests but our service also was struggling to digest all these processes and data fetched.

## Back-pressure? GenStage!

Whenever you need a back-pressure mechanism in Elixir, the answer is obvious, it's `GenStage`. I like the wording from the GenStage [announcement](https://elixir-lang.org/blog/2016/07/14/announcing-genstage):

> In the short-term, we expect GenStage to replace the use cases for GenEvent as well as providing a composable abstraction for consuming data from third-party systems.

This is exactly what we needed: fetching a lot of data from 3rd party service with back-pressure in place.

`GenStage` brings a concept of Producer and Consumer. A Producer has events in its state and Consumer subscribes to the Producer and consumes events according to some rules. `GenStage` comes with a variety of different behaviours for Consumers which dictate the way how events are going to be consumed. Once Consumer is subscribed to Producer, it demands events from Consumer and Producer handles the demand in `handle_demand/2` callback. However, `handle_demand/2` is not the only place from where Producer can send events to Consumer. `handle_call/3`, `handle_info/2` and `handle_cast/2` callbacks have an additional element in the return tuple, so they can send events to Consumer too! Another important detail to note is that once Consumer asked for demand from Producer, it never asks for more demand till it gets all the events it asked previously.

## The implementation

`GenStage` can provide us with back-pressure, but how does it fit with the task at hand? To illustrate that let me introduce steps involved in the processing.

1. fetch Facebook accounts from the database
2. for each Facebook account fetch Facebook Ads accounts from the database
3. for each Facebook Ads account ask Facebook for active campaigns
4. for each Campaign fetch Insights from Facebook API
5. store the fetched data

As you can see, there a lot of repetition of 'for each' statement meaning every one 'event' from the previous step produces more 'events' down the stream. Another important detail to note is that Facebook API has the [quota](https://developers.facebook.com/docs/marketing-api/insights/best-practices) per Facebook account and per Facebook AdsAccount, meaning after eating 100% of the quota Facebook API starts sending errors instead of actual data in the response.

For our purpose, [ConsumerSupervisor](https://hexdocs.pm/gen_stage/ConsumerSupervisor.html) behaviour seemed to be the perfect fit. It works like a pool, but every consumed event has its own separate process. ConsumerSupervisor would restart crashed processes and would demand more events once `min_demand` processes terminate with `:normal` or `:shutdown` status. We could adapt it to our needs, this is how the very beginning of our flow looks like.
<pre>
                                   +------------+
                                   |AdsAccounts |
                              +--->+Producer    |
                              |    |            |
                              |    +------------+
                              |
+----------+      +-----------+    +------------+
| Accounts |      |Accounts   |    |AdsAccounts |
| Producer +<-----+Consumer   +--->+Producer    |
|          |      |Supervisor |    |            |
+----------+      +-----------+    +------------+
                              |
                              |    +------------+
                              |    |AdsAccounts |
                              +--->+Producer    |
                                   |            |
                                   +------------+
</pre>

`AccountsProducer` is a part of the application's Supervisor tree, so it's started when the app is started. It fetches active Facebook accounts from the database and puts them into its state. `AccountsConsumerSupervisor` is also a part of the application's Supervisor tree and it is subscribed to the AccountsProducer. Once `AccountsProducer` gets the Facebook accounts in its state, `AccountsConsumerSupervisor` starts to consume them and spawn a process per each Facebook account consumed. From the code perspective, it looks like the following.

{% raw  %}
```elixir
defmodule Facebook.AccountsProducer do
  use GenStage

  @repopulate_state_interval 1000 * 60

  def start_link do
    GenStage.start_link(__MODULE__, :ok, name: :facebook_accounts_producer)
  end

  def init(:ok) do
    send(self(), :populate_state)

    {:producer, {:queue.new, 0}}
  end

  def handle_demand(incoming_demand, {queue, pending_demand}) do
    {events, state} = dispatch_events(queue, incoming_demand + pending_demand, [])

    {:noreply, events, state}
  end

  def handle_info(:populate_state, {queue, pending_demand}) do
    # Keep in mind this is a strawman implementation.
    # A timer here should be handled with better care: cancelling an old timer before starting a new one.
    Process.send_after(self(), :populate_state, @repopulate_state_interval)

    if :queue.len(queue) < 25 do
      # go to database and fetch facebook accounts
      account_ids = fetch_account_ids()

      queue =
        Enum.reduce account_ids, queue, fn(account_id, acc) ->
          :queue.in(account_id, acc)
        end

      {events, state} = dispatch_events(queue, pending_demand, [])
      {:noreply, events, state}
    else
      {:noreply, [], state}
    end
  end

  # this function stores a pending demand from consumer when there are no
  # events in the state yet, copy-pasted from the GenStage doc
  defp dispatch_events(queue, 0, events) do
    {Enum.reverse(events), {queue, 0}}
  end

  defp dispatch_events(queue, demand, events) do
    case :queue.out(queue) do
      {{:value, event}, queue} ->
        dispatch_events(queue, demand - 1, [event | events])
      {:empty, queue} ->
        {Enum.reverse(events), {queue, demand}}
    end
  end
end
```
{% endraw %}

{% raw %}
```elixir
defmodule Facebook.AccountsConsumerSupervisor do
  use ConsumerSupervisor

  def start_link do
    ConsumerSupervisor.start_link(__MODULE__, :ok, name: :facebook_accounts_consumer_supervisor)
  end

  def init(:ok) do
    children = [
      worker(Facebook.AdsAccountsProducer, [], restart: :transient)
    ]
    opts = [strategy: :one_for_one, subscribe_to: [{:facebook_accounts_producer, max_demand: 25, min_demand: 1}]]
    ConsumerSupervisor.init(children, opts)
  end
end
```
{% endraw %}

The initial number of events to demand and the number of events to trigger for more demand are specified by `max_demand` and `min_demand` options respectively. This allows us to control how many Facebook accounts we would like to process at once. Each `AdsAccountProducer` gets an event (Facebook account_id) from `AccountsProducer`. Once started, `AdsAccountsProducer` fetches from the database all Facebook Ads accounts which belong to given Facebook account and then puts them into its state. `AdsAccountsProducer` uses [Registry](https://hexdocs.pm/elixir/Registry.html) to name processes. Using `Registry` allows us to comply with [Name registration](https://hexdocs.pm/elixir/GenServer.html#module-name-registration) restrictions. Also, we poll `Registry` to report a number of alive workers to the metrics collection system.

{% raw %}
```elixir
defmodule Facebook.AdsAccountsProducer do
  use GenStage

  def start_link(account_id) do
    GenStage.start_link(__MODULE__, account_id, name: name(account_id))
  end

  def name(account_id) do
    {:via, Registry, {:facebook_ads_accounts_producers_registry, account_id}}
  end

  def init(account_id) do
    send(self(), {:fetch_ads_accounts, account_id})

    # meta is a small map with several keys, it holds some useful information
    # like when process was started or facebook account_id
    {:producer, %{demand_state: {:queue.new, 0}, meta: meta}}
  end

  def handle_demand(incoming_demand, %{demand_state: {queue, pending_demand}} = state) do
    {events, demand_state} = dispatch_events(queue, incoming_demand + pending_demand, [])
    state = Map.put(state, :demand_state, demand_state)

    {:noreply, events, state}
  end

  def handle_info({:fetch_ads_accounts, account_id}, %{demand_state: {queue, pending_demand}} = state) do
    # fetch ads_accounts per facebook account_id from database
    ads_accounts = fetch_ads_accounts(account_id)

    queue =
      Enum.reduce ads_accounts, queue, fn(ads_account, acc) ->
        # based on the timestamp when ads_account was processed,
        # determine for which date we should fetch the data from facebook
        dates_to_fetch = calculate_dates_to_fetch(ads_account)
        Enum.reduce dates_to_fetch, acc, fn(date, acc_2) ->
          :queue.in({ads_account, date}, acc_2)
        end
      end

    {events, demand_state} = dispatch_events(queue, pending_demand, [])

    state = Map.put(state, :demand_state, demand_state)

    {:noreply, events, state}
  end
end

```
{% endraw %}

Great, now we can 'produce' and 'consume' Facebook accounts, but what's next? Each `AdsAccountsProducer` holds some AdsAccounts in its state, but there are no consumers which would consume them to continue the flow. So why not to spawn consumers dynamically per `AdsAccountProducer` and use the same `ConsumerSupervisor` logic further?
<pre>
                                                                            +------------+
                                                                            |Campaigns   |
                                                                      +---> |Producer    |
                                                                      |     |            |
                                                                      |     +------------+
                                                                      |
                                      +------------+      +-----------+     +------------+
                                      |AdsAccounts |      |AdsAccounts|     |Campaigns   |
                                +---> |Producer    | <----+Consumer   +-->  |Producer    |
                                |     |            |      |Supervisor |     |            |
                                |     +------------+      +-----------+     +------------+
                                |                                     |
+-----------+       +-----------+     +------------+                  |     +------------+
| Accounts  |       |Accounts   |     |AdsAccounts |                  |     |Campaigns   |
| Producer  | <-----+Consumer   +-->  |Producer    |                  +---> |Producer    |
|           |       |Supervisor |     |            |                        |            |
+-----------+       +-----------+     +------------+                        +------------+
                                |
                                |     +------------+
                                |     |AdsAccounts |
                                +---> |Producer    |
                                      |            |
                                      +------------+
</pre>

Starting consumer dynamically would require adding `AdsAccountsConsumerSupervisor.start_link(account_id, self())` to `handle_info/2` in `AdsAccountsProducer`, so it would start a consumer for itself after it puts AdsAccounts into its state. The `self()` among the arguments is required so `AdsAccountsConsumerSupervisor` knows a process it needs to subscribe to.

{% raw %}
```elixir
defmodule Facebook.AdsAccountsConsumerSupervisor do
  use ConsumerSupervisor

  def start_link(account_id, pid_to_subscribe) do
    name = name(account_id)
    ConsumerSupervisor.start_link(__MODULE__, {pid_to_subscribe, account_id}, name: name)
  end

  def name(account_id) do
    {:via, Registry, {:facebook_ads_accounts_consumer_supervisors_registry, account_id}}
  end

  def init({pid_to_subscribe, account_id}) do
    children = [
      worker(Facebook.CampaignsProducer, [], restart: :transient)
    ]
    opts = [strategy: :one_for_one, subscribe_to: [{pid_to_subscribe, max_demand: 10, min_demand: 1}]]
    ConsumerSupervisor.init(children, opts)
  end
end
```
{% endraw %}

Every `AdsAccountProducer` starts its own consumer, which would consume AdsAccounts and spawn `CampaignProducer` per each Facebook AdsAccount. `CampaignsProducer` gets AdsAccount and a date to fetch, then it asks Facebook API for active campaigns which are running under given AdsAccount for given date. And then finally it puts campaigns into its state and, you guessed it, starts a consumer for itself.
<pre>
                                                                                                                +------------+
                                                                                                                |Insights    |
                                                                                                          +---> |Producer    |
                                                                                                          |     |            |
                                                                                                          |     +------------+
                                                                                                          |
                                                                           +------------+     +-----------+     +------------+
                                                                           |Campaigns   |     |Campaigns  |     |Insights    |
                                                                     +---> |Producer    <-----+Consumer   +-->  |Producer    |
                                                                     |     |            |     |Supervisor |     |            |
                                                                     |     +------------+     +-----------+     +------------+
                                                                     |                                    |
                                      +------------+     +-----------+     +------------+                 |     +------------+
                                      |AdsAccounts |     |AdsAccounts|     |Campaigns   |                 |     |Insights    |
                                +---> |Producer    <-----+Consumer   +-->  |Producer    |                 +---> |Producer    |
                                |     |            |     |Supervisor |     |            |                       |            |
                                |     +------------+     +-----------+     +------------+                       +------------+
                                |                                    |
+-----------+       +-----------+     +------------+                 |     +------------+
| Accounts  |       |Accounts   |     |AdsAccounts |                 |     |Campaigns   |
| Producer  | <-----+Consumer   +---> |Producer    |                 +---> |Producer    |
|           |       |Supervisor |     |            |                       |            |
+-----------+       +-----------+     +------------+                       +------------+
                                |
                                |     +------------+
                                |     |AdsAccounts |
                                +---> |Producer    |
                                      |            |
                                      +------------+
</pre>

Every `InsightProducer` gets a Facebook `campaign_id`, fetches Insights from Facebook Marketing API and puts the fetched data into its state.

Unfortunately, `InsightsProducer` cannot store data yet. A Facebook campaign's insights represents data per day, whereas at Adjust we have to store this data per hour because of timezones support. Therefore a Consumer for `InsightsProducer` needs to issue an additional HTTP request to Facebook API for every Ad to get hourly distribution. The fact that we have quite a lot of Ads to ask hourly distribution for imposes some limitations to the way how we can consume Insights from every `InsightsProducer`. Consuming the events from `InsightsProducer` the same way using ConsumerSupervisor behaviour would generate a lot of concurrent requests to Facebook even if `max_demand` would be 2, so quota would be consumed quite fast. Therefore the Consumer for `InsightsProducer` should consume events slowly and check quota after every request. Fortunately, GenStage comes with `manual` mode, which allows consuming events explicitly. Once a Consumer is set into `manual` mode, there is no `max_demand` and `min_demand` anymore, one should ask for events explicitly instead.

<pre>
                                                                                                                +----------+    +----------+    +----------+
                                                                                                                |Insights  |    |CostData  |    |CostData  |
                                                                                                          +---> |Producer  <----+Producer  <----+Consumer  |
                                                                                                          |     |          |    |Consumer  |    |          |
                                                                                                          |     +----------+    +----------+    +----------+
                                                                                                          |
                                                                           +------------+     +-----------+     +----------+    +----------+    +----------+
                                                                           |Campaigns   |     |Campaigns  |     |Insights  |    |CostData  |    |CostData  |
                                                                     +---> |Producer    <-----+Consumer   +-->  |Producer  <----+Producer  <----+Consumer  |
                                                                     |     |            |     |Supervisor |     |          |    |Consumer  |    |          |
                                                                     |     +------------+     +-----------+     +----------+    +----------+    +----------+
                                                                     |                                    |
                                      +------------+     +-----------+     +------------+                 |     +----------+    +----------+    +----------+
                                      |AdsAccounts |     |AdsAccounts|     |Campaigns   |                 |     |Insights  |    |CostData  |    |CostData  |
                                +---> |Producer    <-----+Consumer   +---> |Producer    |                 +---> |Producer  <----+Producer  <----+Consumer  |
                                |     |            |     |Supervisor |     |            |                       |          |    |Consumer  |    |          |
                                |     +------------+     +-----------+     +------------+                       +----------+    +----------+    +----------+
                                |                                    |
+-----------+       +-----------+     +------------+                 |     +------------+
| Accounts  |       |Accounts   |     |AdsAccounts |                 |     |Campaigns   |
| Producer  | <-----+Consumer   +---> |Producer    |                 +---> |Producer    |
|           |       |Supervisor |     |            |                       |            |
+-----------+       +-----------+     +------------+                       +------------+
                                |
                                |     +------------+
                                |     |AdsAccounts |
                                +---> |Producer    |
                                      |            |
                                      +------------+
</pre>

`CostDataProducerConsumer` is set to manual mode, it's started by `InsightsProducer`, demands one event (one ad), sends a request to Facebook API, gets the data and passes it to the `CostDataConsumer` which finally stores it to the database. After every request to the Facebook API, `CostDataProducerConsumer` checks quota values in the response headers: if the quota is nearly depleted, it demands a new event from `InsightsProducer` with some delay using `Process.send_after/3`. Otherwise, if the quota values allow, it does that immediately. Also, a Consumer of InsightsProducer is actually a ProducerConsumer, because it both consumes and produces events. Here is how one can set a Consumer or ProducerConsumer into the `manual` mode.

{% raw %}
```elixir
defmodule Facebook.CostDataProducerConsumer do
  use GenStage

  # subscribe_to_pid is PID of InsightsProducer
  def start_link(apps, ads_account, campaign_id, date, subscribe_to_pid) do
    GenStage.start_link(__MODULE__, {apps, ads_account, campaign_id, date, subscribe_to_pid}, name: name(ads_account.id, campaign_id, date))
  end

  def name(ads_account_id, campaign_id, date) do
    {:via, Registry, {:facebook_cost_data_producers_registry, {ads_account_id, campaign_id, date}}}
  end

  def init({apps, ads_account, campaign_id, date, subscribe_to_pid}) do
    Facebook.CostDataConsumer.start_link(ads_account, campaign_id, date, self())

    # state would contain some meta info
    state = %{campaign_id: campaign_id, ads_account: ads_account, date: date}

    # note that there is no max_demand or min_demand here
    {:producer_consumer, state, subscribe_to: [subscribe_to_pid]}
  end

  # a callback is invoked when this producer_consumer is subscribed to InsightsProducer
  def handle_subscribe(:producer, _opts, from, state) do
    # once this consumer is subscribed, ask InsightsProducer for one event
    GenStage.ask(from, 1)

    state = Map.put(state, :from, from)

    {:manual, state}
  end

  # a callback is invoked when CostDataConsumer is subscribed to this ProducerConsumer
  # CostDataConsumer would consume events in automatic mode asking for max_demand events
  def handle_subscribe(:consumer, _opts, _from, state) do
    {:automatic, state}
  end

  # here is super simplified version of `handle_events/3` callback
  #
  # we always ask for only one event, that's why [event] here
  def handle_events([event], {pid, _sub_tag}, state) do
    # send a request to fb api
    {hourly_distribution, quota} = fetch_hourly_distribution(event, state)

    # apply hourly distribution to the daily data
    events = apply_hourly_distribution(event)

    # demand new event with a possible timeout
    demand_event_maybe(quota)

    # and pass `events` down the flow to the CostDataConsumer
    {:noreply, events, state}
  end

  # a callback to ask one more event from InsightsProducer
  # the pid of InsightsProducer is put into state in `handle_subscribe/4` callback
  def handle_info(:demand_event, state) do
    GenStage.ask(state.from, 1)
  end

  # if quota is too high, ask an event a bit later
  defp demand_event_maybe(quota) when quota > 90 do
    Process.send_after(self(), :demand_event, :timer.minutes(2))
  end

  defp demand_event_maybe(_), do: send(self(), :demand_event)
end
```
{% endraw %}

That is finally the end of the flow. So far it looks like the following:

1. `AccountsProducer` is started by the main Supervisor, gets accounts from db, puts into its state
2. `AccountsConsumerSupervisor` is started by the main Supervisor, it subscribes to `AccountsProducer`, consumes events (accounts) and spawn one `AdsAccountProducer` per each account
3. Each `AdsAccountsProducer` fetches Facebook account's AdsAccounts from the database, puts them into state and starts dynamically a ConsumerSupervisor for itself
4. `AdsAccountsConsumerSupervisor` consumes AdsAccounts, spawns one `CampaignsProducer` per each AdsAccount
5. Each `CampaignsProducer` gets AdsAccount, fetches active campaigns from Facebook API, puts them into its state and starts a `CampaignsConsumerSupervisor` for itself
6. `CampaignsConsumerSupervisor` consumes campaigns, spawns one `InsightsProducer` per each campaign
7. Each `InsightsProducer` gets campaign's Insights from Facebook API, puts the data into its state and starts a consumer for itself
8. A consumer for `InsightsProducer` is `CostDataProducerConsumer`, it's set into `manual` mode and consumes events one by one, for every consumed event (an ad) it sends additional HTTP request to Facebook API, gets the data and passes it further to `CostDataConsumer`
9. `CostDataConsumer` gets all the data, does some transformations (timezone conversion, currency conversion, etc) and puts data into the database

Phew. That's a lot happening here, but although it might look complicated, in fact, the architecture is quite simple. The same ConsumerSupervisor behaviour was applied several times to run multiple Facebook Accounts, AdsAccounts and Campaigns processes concurrently and without blocking each other.

Now, the question is how and when a producer process exits with `:normal` or `:shutdown` status, so ConsumerSupervisors can demand more events and spawn more processes. So let's follow the termination path, i.e. how these GenStage processes get terminated. Let's start with the last part: `InsightsProducer` - `CostDataProducerConsumer` - `CostDataConsumer`. `CostDataProducerConsumer` demands events from `InsightsProducer` one by one and passes the events down the flow to the `CostDataConsumer.` Every time an event is consumed,  `CostDataProducerConsumer` asks its `InsightsProducer` how many events are left in its state. When the answer is zero, `CostDataProducerConsumer` sends an event to `CostDataConsumer` indicating that there was the last event. Let's see how it would be implemented in `handle_events/3` callback from the listing above.

{% raw %}
```elixir
defmodule Facebook.CostDataProducerConsumer do
  use GenStage

  def handle_events([event], {pid, _sub_tag}, state) do
    insights_queue_len = GenServer.call(pid, :get_queue_len)

    # send a request to fb api
    {hourly_distribution, quota} = fetch_hourly_distribution(event, state)

    # apply hourly distribution to the daily data
    events = apply_hourly_distribution(event)

    if insights_queue_len == 0 do
      cost_data_consumer_pid =
        Facebook.CostDataConsumer.name(state.ads_account.id, state.campaign_id, state.date)
        |> GenServer.whereis()

      # if there are no events left in InsightsProducer, no need to demand more events
      # just tell the CostDataConsumer that these `events` are the last ones
      if cost_data_consumer_pid, do: send(cost_data_consumer_pid, :last_event)
    else
      # demand new event with a possible timeout
      demand_event_maybe(quota)
    end

    {:noreply, events, state}
  end
end
```
{% endraw %}

After that `CostDataConsumer` has 10 seconds to finish processing and storing the last batch of the events. After 10 seconds it terminates itself with `:normal` status.

{% raw %}
```elixir
defmodule Facebook.CostDataConsumer do
  use GenStage

  @ttl_interval Application.get_env(:settings, :facebook)[:ttl_interval]

  def start_link(ads_account, campaign_id, date, subscribe_to_pid) do
    GenStage.start_link(__MODULE__, {ads_account, campaign_id, date, subscribe_to_pid}, name: name(ads_account.id, campaign_id, date))
  end

  def name(ads_account_id, campaign_id, date) do
    {:via, Registry, {:facebook_cost_data_consumers_registry, {ads_account_id, campaign_id, date}}}
  end

  def init({ads_account, campaign_id, date, subscribe_to_pid}) do
    state = %{
      ads_account: ads_account,
      campaign_id: campaign_id,
      date: date
    }
    {:consumer, state, subscribe_to: [{subscribe_to_pid, subscription_opts()}]}
  end

  # simplified version of `handle_events/3` callback
  def handle_events(events, _from, state) do
    Enum.each(events, fn(event) -> store_data(event) end)

    {:noreply, [], state}
  end

  # CostDataConsumer gets this event from `CostDataProducerConsumer` and sends another one
  # to itself to terminate itself with `:normal` state.
  def handle_info(:last_event, state) do
    Process.send_after(self(), :die, @ttl_interval)

    {:noreply, [], state}
  end

  def handle_info(:die, state) do
    {:stop, :normal, state}
  end
end
```
{% endraw %}

Since `CostDataConsumer`, `CostDataProducerConsumer` and `InsightsProducer` are linked using `start_link/3`, termination of `CostDataConsumer` would terminate `InsightsProducer` and `CostDataProducerConsumer` with the same status. Once `InsighsProducer` goes down with `:normal` state, `CampaignsConsumerSupervisor` can demand more campaigns from `CampaignsProducer` and spawn more `InsightsProducers` for the new campaigns.

Now let's see how `CampaignsProducer` and `AdsAccountsProducer` terminate itself. The logic is the same for both of these producers, so let me show in detail how `CampaignsProducer` exits with `:normal` state. `CampaignsProducer` checks its state every 5 seconds and when there are no more campaigns in its state to process and there are no `InsightsProducers` active, it exits with `:normal` state, which allows `AdsAccountsConsumerSupervisor` to spawn more `CampaignsProducer` for the newly consumed AdsAccounts.

{% raw %}
```elixir
defmodule Facebook.CampaignsProducer do
  use GenStage

  @heartbeat_interval Application.get_env(:settings, :facebook)[:heartbeat_interval]

  def start_link({ads_account, date}) do
    GenStage.start_link(__MODULE__, {ads_account, date}, name: name(ads_account.id, date))
  end

  def name(ads_account_id, date) do
    {:via, Registry, {:facebook_campaigns_producers_registry, {ads_account_id, date}}}
  end

  def init({ads_account, date}) do
    send(self(), {:fetch_campaigns, ads_account, date})

    Process.send_after(self(), :die_maybe, @heartbeat_interval)

    meta = %{
      active: false,
      ads_account: ads_account,
      date: date
    }

    {:producer, %{demand_state: {:queue.new, 0}, meta: meta}}
  end

  # handle_demand/3 and other callbacks are omitted

  # every `@heartbeat_interval` seconds (5 by default) it checks the its state
  # the `active` key in meta is set to `true` once state is populated
  # this is done to not shutdown CampaignsProducers which haven't had a chance to fetch
  # campaigns from facebook api yet
  def handle_info(:die_maybe, %{demand_state: {queue, _}, meta: meta} = state) do
    if :queue.len(queue) == 0 and meta.active and not consumers_alive?(meta.ads_account.id, meta.date) ->
      {:stop, :normal, state}
    else
      # Keep in mind this is a strawman implementation.
      # A timer here should be handled with better care: cancelling an old timer before starting a new one.
      Process.send_after(self(), :die_maybe, @heartbeat_interval)
      {:noreply, [], state}
    end
  end

  defp consumers_alive?(ads_account_id, date) do
    consumer_sup_name = Facebook.CampaignsConsumerSupervisor.name(ads_account_id, date)

    with consumer_pid when not is_nil(consumer_pid) <- GenServer.whereis(consumer_sup_name),
         %{active: active} <- ConsumerSupervisor.count_children(consumer_pid) do
      active > 0
    else
      _ ->
        false
    end
  end

```
{% endraw %}

`AdsAccountsProducer` has the same logic, the only difference is ConsumerSupervisor name in `consumers_alive?/2` function.


The only GenStage processes which never goes down (unless there is an exception) are `AccountsProducer` and `AccountsConsumerSupervisor`. Once the number of accounts in `AccountsProducer` is closing to zero, it repopulates its state with more accounts from the database, so it never stops producing events.

## Summary

`GenStage` allows a developer to build sophisticated data flows with back-pressure in place. It provides necessary abstractions for producing and consuming events. In combination with `Registry`, we could build a robust application which can fetch and process Facebook cost data for thousands of different AdsAccount without blocking each other. Every AdsAccount, Campaigns or Ad is processed separately from each other and if any of processes crashes, GenStage's ConsumerSupervisor would restart it. The application can dynamically speed up or slow down the flow by itself based on Facebook quota values.

This blog post got long enough already and I even haven't started to talk about one of the most important part of the application — HTTP client. We send over 6 millions of heavy, long-lasting HTTP requests to Facebook per day, so having a reliable and fast HTTP client is vital. This is going to be a topic for my next blog post. Stay tuned!
