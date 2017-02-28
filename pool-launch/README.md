# Kings of Pool Launch

Kings of Pool was world wide launched on Tuesday, Feb 21st, 2017. On Thursday, Feb 23rd, 2017 it started to receive features from Android's Google Play and Apple's App Stores in many countries. These features have substantially increase the number of online users we received and added a lot of stress to several components of the backend.

![Users online](users_online.png)

## Disasters

There were 4 Disasters but only 2 caused total downtime. The Redis issue caused a short outage of 10 minutes, the HAProxy outage was more serious and caused the game to be down from 2am to 7am.

24/02/2017

- [Redis - 8am](Redis)
- [Flubsub - 7am](Flubsub)

25/02/2018

- [HAproxy - 2am ~ 7am](HAProxy)
- [Vuvuzela - 8am](Vuvuzela)

### Redis - 24/02/2017 @ 8am

Pool production redis is configured to use AWS Elasticache, an instance with RAM size of 3GB. Pool uses redis in different ways, one of the main uses is as an in memory storage for a Replay Attack prevention solution.

Almost every request made to Pool's backend creates a key in redis and this key has a TTL. The combination of number of requests + redis server memory size(3gb) + long TTL of 5 hours and the verbosity of the HTTP pooling solution caused redis to quickly reach its memory capacity. Redis has expiry policies (e.g.: LRU) for when the memory limit is reached and this shouldn't cause an issue, but it still requires some tweaking.

The crippling error that we observed was:
```
Redis::CommandError: MISCONF Errors writing to the AOF file: No space left on device
```

AWS Elasticache doesn't let you know the disk size on this instances (It's hidden from the user) and you would hope that they provision enough disk space to account for AOF and RDB files. Since this didn't seem to be the case we hoped that an instance with double the size (as in RAM) would also have a larger disk.

Watching redis keys being set

Before (5 hours TTL)
```
1487973821.158271 [0 10.0.0.159:45462] "expire" "authenticate:187203:19i6gdp8dgCZawqN3hkHuahaihI=" "18000"
```
After (20 minutes TTL)
```
1487973821.158271 [0 10.0.0.159:45462] "expire" "authenticate:187203:19i6gdp8dgCZawqN3hkHuahaihI=" "1200"
```
Before, used memory was over 3GB, 5 hours later (After the old keys had time to expire):
```
admin@jumpbox:~$ redis-cli -h pool-redis.uken.int info | grep used_memory_human | cut -d: -f2 | awk '{print "pool.redis.usedmemory "$1}'
pool.redis.usedmemory 979.46M
```

#### Fixes

- Doubled the redis instance size to have 6GB RAM.
- Excluded the verbose messages endpoint from going to redis(no need for replay attack prevention here):  [FIX](https://github.com/uken/pool-rails/commit/542e1331ba46d4e3f7d11f2427a2575f271cd024)
- Lowered TTL from 5 hours to 20 minutes: [FIX](https://github.com/uken/pool-rails/pull/635/files)
- Added newrelic redis tracing: [FIX](https://github.com/uken/pool-rails/pull/633)

#### Result

After a few hours we observed redis memory usage to actually go down and stabilize as opposed to growing "unbounded".

- 4:47pm: used_memory_human:3.62G
- 5:03pm: used_memory_human:3.49G


### Flubsub - 24/02/2017 @ 7am
#### Flubsub response times
![](flubsub.png)

### HAProxy - 25/02/2017 @ 2am

Duration: 5 hours

There was a catastrophic failure where all 4 instances of HAProxy failed at the same time. The HAProxy instances only use init.d scripts instead of proper process managers and for that reason they didn't restart. The reason for all 4 instances to fail simultanously is unknown but it caused all our legacy infrastructure that is load balanced by HAProxy (and the apps that depend on them) to fail. This has brought down USys which in turn brought down Pool Rails backend servers.

#### Alerting
It's very concerning that no alerts were triggered at 2am when all HAProxy instances failed:
![no alerts](no-alerts.png)
This means that the HAProxy instances themselves didn't trigger alerts and also all the affected infrastructure didn't trigger alerts. We are still investigating how this could have happened as many of these affected apps do have alerting setup.

You can see that an alert was manually trigger at 5:20am to summon help to deal with the issue.

#### Usys Outage
![](usys.png)

Other apps that relied on HAProxy (including USys and Kings of Pool rails backend since it depends on Usys) experienced a similar disruption from 2am to 7am.

#### Fix
The fix was to simply restart the HAProxy instances.
### Vuvuzela - 25/02/2017 @ 8am
#### Vuvuzela Error rate
![](vuvuzela.png)
