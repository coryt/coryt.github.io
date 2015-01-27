---
layout: post
title: API Throttling with ServiceStack
excerpt: ""
tags: [servicestack, redis, api]
modified: 2015-01-25
share: false
comments: true
---

Today I'd like to show a little plugin I wrote to enable throttling on any of my api endpoints. ServiceStack is my framework of choice and we use it at Kobo to serve up our client APIs. If you aren't already familiar with ServiceStack, head on over and check it out[^1]. I couldn't be happier with ServiceStack, it's an incredibly fast and flexible framework and currently meets all of our needs. So let's jump right in. 

##So Why Throttle Clients?
There may be many business-driven reasons why one would want to add throttling to an api. In a lot of cases, throttling is intented to prevent abuse of an API whether that abuse is intentional or not, it's always a good idea to ensure your apis are protected and meet your defined SLAs. Other cases maybe to control costs, limit an attack vector, think an attacker brute forcing an authentication endpoint, and so on.

##What can be Throttled?
ProgrammableWeb[^2] had a good summary of the many ways companies are limiting their apis and it's not just request per ip over a specific time period. A few examples are:
* Time based limits: 1 call per second
* Call volume by IP: 5,000 queries per IP per day
* Call volume per-application: 10,000 queries per application per day
* Return results volume: 10 results per query
* Data transmission volume: 120 packets of 1.6KB per minute

So with that, our goal for this post is to easily enable specific ServiceStack endpoints to monitor and throttle requests by clients at per minute, per hour or per day intervals.

##Annotating our API Endpoints
In order to let our plugin know which endpoints we want to throttle, we need a way to specificly mark each ServiceStack Operation. We'll do that with the ThrottleInfoAttribute[^5] class and example usage below.

{% highlight c# linenos %}
public class ThrottleInfoAttribute : Attribute
{
    public int PerMinute { get; set; }
    public int PerHour { get; set; }
    public int PerDay { get; set; }
}
{% endhighlight %}

{% highlight c# linenos %}
[ThrottleInfo(PerMinute = 10, PerHour = 0, PerDay = 0)]
[Route("/perminute")]
public class GetPerMinuteThrottle : IReturn<string>
{

}
{% endhighlight %}

##The ServiceStack Plugin
Don't worry if you aren't familiar with writing a plugin for ServiceStack, their wiki[^3] has a pretty thorough walkthrough and you can also view the full code for the plugin[^4]. 

The Register function is called when ServiceStack loads all plugins, at which point we want our plugin to lookup all of the Operations with a ```ThrottleInfoAttribute``` so that we can cache the results and we don't have to do it on each request. 

{% highlight c# linenos %}
public void Register(IAppHost appHost)
{
    RegisterThrottleInfoForAllOperations(appHost);

    _redisClient.RemoveAllLuaScripts();
    _scriptSha = _redisClient.LoadLuaScript(ReadLuaScriptResource("rate_limit.lua"));

    appHost.GlobalRequestFilters.Add(CheckIfRequestShouldBeThrottled);
}
{% endhighlight %}

{% highlight c# linenos %}
public void RegisterThrottleInfoForAllOperations(IAppHost appHost)
{
    foreach (var operation in appHost.Metadata.Operations)
    {
        var throttleAttribute = operation.RequestType.GetCustomAttributes(typeof(ThrottleInfoAttribute)).First() as ThrottleInfoAttribute;
        if (throttleAttribute == null)
            continue;

        _throttleInfoMap.Add(operation.RequestType, throttleAttribute);
    }
}
{% endhighlight %}

We also want to optimize our calls to Redis so we should pre-load our Lua script into Redis so as to avoid sending this on each request.

{% highlight c# linenos %}
_redisClient.RemoveAllLuaScripts();
_scriptSha = _redisClient.LoadLuaScript(ReadLuaScriptResource("rate_limit.lua"));
{% endhighlight %}

Next is our function ```CheckIfRequestShouldBeThrottled``` which is called on every request. We can do a quick lookup to see if our request type has throttling info which, if it does, build up a key of RemoteIp and OperationName. This will be used to create buckets based on the time internal configured for this Operation. We'll touch more on this shortly. Then we actually need to call into redis, using the sha we received earlier to the cached lua script, the key and the throttling information for the given Operation being called. Our script returns 1 if we should throttle the request or null if we shouldn't, so check the result, and if we are supposed to throttle this request, set the status to 429 with a description that the client has been throttled.

{% highlight c# linenos %}
private void CheckIfRequestShouldBeThrottled(IRequest request, IResponse response, object requestDto)
{
    if (!_throttleInfoMap.ContainsKey(requestDto.GetType()))
        return;

    var throttleInfo = _throttleInfoMap[requestDto.GetType()];
    var key = string.Format("{0}:{1}", request.RemoteIp, request.OperationName);

    try
    {
        var result = _redisClient.ExecLuaShaAsString(_scriptSha, new[] {key}, new[]
        {
            throttleInfo.PerMinute.ToString(), 
            throttleInfo.PerHour.ToString(), 
            throttleInfo.PerDay.ToString(),
            SecondsFromUnixTime().ToString()
        });
        if (result != null)
        {
            response.StatusCode = 429;
            response.StatusDescription = "Too many Requests. Back-off and try again later.";
            response.Close();
        }
    }
    catch ()
    {
        //got an error calling redis so log something and let the request through
    }
}
{% endhighlight %}

##Determing if we should Throttle
The benefit of using a script directly on the redis server is that we can reduce the number of roundtrips we need to make. Everything can be done in one call to redis. Above you'll note we are passing into the script the key of ```ip:operation``` followed by four arguments (3 limits and a timestamp). The last argument is the timestamp, which we remove from the table and assign to ts. Next we loop over the lesser of the number of arguments or the number of durations. We build up our bucket key which we call ```INCR``` on and set it's expiry to the duration window of 60sec, 3600sec or 86400sec. This ensures the bucket will expire at the end of this period or we'll keep incrementing the counter until we hit the limit.

{% highlight lua linenos %}
local durations = {60, 3600, 86400}
local prefix    = {'m', 'h', 'd'}
local ts        = tonumber(table.remove(ARGV))
local request   = KEYS[1]

--iterate over all of the limits provided
for i = 1, math.min(#ARGV, #durations) do
	local limit = tonumber(ARGV[i])

	--only check limits that have been set
    if limit > 0 then
	    local bucket = ':' .. prefix[i] .. ':' .. durations[i] .. ':' .. math.floor(ts / durations[i])
	    local key = request .. bucket

	    local total = tonumber(redis.call('INCR', key) or '0')
	    redis.call('EXPIRE', key, durations[i])
	    if total > limit then
	        return true
	    end
    end
end
return false
{% endhighlight %}

The main goal of this post was to show how we could write a plugin for ServiceStack to enable throttling per operation so I've kept the implementation simple enough to follow even if you aren't familiar with lua. You can check out the post on stack exchange[^7] which lead me to explore using lua in the first place or read antirez blog[^6] for some background on redis and lua scripting. 

--------
[^1]: <http://servicestack.net>
[^2]: <http://www.programmableweb.com/news/12-ways-to-limit-api/2007/04/02>
[^3]: <https://github.com/ServiceStack/ServiceStack/wiki/Plugins>
[^4]: <https://gist.github.com/coryt/f1d040047cc45b6650ff#file-throttleplugin-cs>
[^5]: <https://gist.github.com/coryt/d07e95b403c4996600b8#file-throttleinfoattribute-cs>
[^6]: <http://oldblog.antirez.com/post/redis-and-scripting.html>
[^7]: <http://codereview.stackexchange.com/questions/30195/redis-rate-limiting-in-lua>