# Redis on Wodby

What Wodby sets up for this Redis service. Check it before adding Redis connection settings or a Redis configuration file to an application.

## How applications reach it

- Host: the name of this app service inside the environment. Port: `6379`.
- A password is always required. Wodby generates it (token `password`) and the server starts with `requirepass`.
- A service linked to this one receives the host, port and password as environment variables. The names are chosen by the linking service, for example `REDIS_HOST`, `REDIS_PORT` and `REDIS_PASSWORD`, or one `REDIS_URL`. Read those variables in code; do not hardcode the host or copy the password into the repository.
- Inside this container the password is in `REDIS_PASSWORD`.

## Generated configuration

On every start the container renders `/etc/redis.conf` from its environment variables and starts `redis-server` with it. The file is rewritten on each start: never edit it. Configuration is changed with environment variables on this service, and applies with the next deployment.

| Variable | Effect | Image default |
| --- | --- | --- |
| `REDIS_MAXMEMORY` | `maxmemory` | `128m` |
| `REDIS_MAXMEMORY_POLICY` | `maxmemory-policy` | `allkeys-lru` |
| `REDIS_DATABASES` | number of databases | `16` |
| `REDIS_TIMEOUT` | idle client timeout, seconds | `300` |
| `REDIS_SAVE_TO_DISK` | any value turns persistence on | set by Wodby when the data volume exists |
| `REDIS_APPENDONLY`, `REDIS_APPENDFSYNC`, `REDIS_SAVES` | AOF and RDB settings, used only with persistence on | `yes`, `everysec`, `900:1/300:10/60:10000` |
| `REDIS_NOTIFY_KEYSPACE_EVENTS` | keyspace notifications | empty |

## Eviction and persistence

- With the defaults this is a cache: once `REDIS_MAXMEMORY` is reached, any key can be evicted (`allkeys-lru`), including keys without a TTL.
- A job queue, a session store or any data that must not disappear needs `REDIS_MAXMEMORY_POLICY` set to `noeviction` and enough `REDIS_MAXMEMORY`. Using the default cache settings as a queue store is the usual mistake: jobs are dropped silently under memory pressure.
- The `data` volume is optional. Without it nothing is written to disk (`save ""`, no AOF) and every restart or deployment starts empty. With it, Wodby turns persistence on and data is kept in `/data` (RDB snapshots plus AOF).
- The manifest declares no backups, imports or scheduled jobs for this service.

## Check the result

From this service's container:

- `redis-cli -a "$REDIS_PASSWORD" ping` answers `PONG`.
- `redis-cli -a "$REDIS_PASSWORD" config get maxmemory-policy` and `config get appendonly` show the eviction policy and whether persistence is on.
- `redis-cli -a "$REDIS_PASSWORD" info keyspace` shows which databases hold keys; `info stats` has `evicted_keys`.
