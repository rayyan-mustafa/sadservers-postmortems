# Bern: WordPress could not find its own database

| | |
|---|---|
| **Scenario** | [Bern: Docker WordPress and MariaDB](https://sadservers.com/scenario/bern) |
| **Difficulty** | Hard |
| **Skills** | Docker networking, default bridge vs user-defined network, container DNS |
| **Date solved** | Nov 2026 (14 minutes, 0 clues) |

## Symptom

Two Docker containers were running on the box: a WordPress site and a MariaDB database. Opening `localhost:80` showed "Error establishing a database connection". The WordPress code itself was fine. The task was to make WordPress reach MariaDB using the hostname `mysql`, user `root` and password `password`. The pass condition was `docker exec wordpress mysqladmin -h mysql -u root -ppassword ping` answering "mysqld is alive".

## Investigation

| # | Command | What it told me |
|---|---|---|
| 1 | `sudo docker ps` | Both containers were up: `wordpress` and `mariadb`. So the database was not down. Note the database container is called `mariadb`, but the app expects the host `mysql`. |
| 2 | `sudo docker inspect -f '{{.Name}} -> {{range $k, $v := .NetworkSettings.Networks}}{{$k}} {{end}}' wordpress mariadb` | Both containers were attached only to the default `bridge` network (`172.17.0.2` and `172.17.0.3`). |
| 3 | `sudo docker network ls` | The list of Docker networks, to see what I could attach the containers to. Both containers were only on the default `bridge`, so I made a new network. |

**Dead ends**

- **Credentials.** The task gave `root` / `password`, and the database was up, so this was not an authentication problem.
- **A dead database.** `docker ps` showed MariaDB running, so I stopped looking at the database itself.
- **Assuming the container name would be enough.** Even on a user-defined network, the name `mariadb` resolves but `mysql` does not. The app asks for `mysql`, so the name had to be added as an alias.

## Root cause

Two separate things, and both had to be fixed:

1. `wordpress` and `mariadb` were both on Docker's default bridge network. On that network containers can reach each other by IP address only. There is no automatic DNS by container name.
2. The database container was named `mariadb`, but WordPress connects to the host `mysql`. Even with working name resolution, `mysql` would not have matched anything.

## Fix

```bash
# create a user-defined network (containers get DNS by name on these)
sudo docker network create wp-net
sudo docker network connect wp-net wordpress

# connect the database and give it the extra name the app expects
sudo docker network connect --alias mysql wp-net mariadb

# check
sudo docker exec wordpress mysqladmin -h mysql -u root -ppassword ping   # "mysqld is alive"
curl -s localhost:80 | tail -4
```

How I verified: the `mysqladmin ping` test is the pass condition, and I then ran "Check My Solution" on SadServers, which accepted it.

## Prevention

- Do not rely on the default bridge for anything that needs to talk by name. Put related containers on one user-defined network, ideally through Docker Compose, which does this for you.
- Keep the database service name and the app's `WORDPRESS_DB_HOST` setting in one place (the Compose file), so they cannot drift. Either name the service `mysql`, or set the host to the real service name (`mariadb`).
- If you do need an extra name for an existing container, `docker rename` or a network alias is a one-line change. Write it down in the Compose file so the next person can see it.

## What I learned

On the default bridge a container name is just a label; names only resolve on a user-defined network, and the name has to match what the app asks for.
