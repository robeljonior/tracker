# Tracker backend

A personal find-my-device system: a Spring Boot API and a Go agent for Linux.
Needs Java 21, Maven and Go 1.22.

## 1. Start the backend

```bash
mvn package
```

```bash
java -XX:+UseSerialGC -Xmx96m -XX:TieredStopAtLevel=1 -jar target/tracker-backend-0.2.0.jar
```

It listens on port 8080 and keeps its data in `./data`. During development,
`mvn spring-boot:run` does the same with the same memory settings.

The first start creates your **owner key** in `data/owner-key`. Enter it in
tracker-web's Settings. To choose your own, set `OWNER_API_KEY` (16+ characters).

## 2. Connect the laptop

```bash
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o bin/tracker-agent ./cmd/linux-agent
```

```bash
./bin/tracker-agent -pair -server http://LAPTOP_IP:8080 -device-id linux-pc-01 -name "My Linux PC" -install-autostart
```

The agent opens a QR code. Scan it with your phone, or enter the code it prints in
tracker-web, and approve it with your owner key. Use the laptop's network address
rather than `localhost`, so the phone can open the page.

Settings and the device key are saved in `~/.config/tracker-agent/config.json`, so later
runs need no flags. If the backend ever rejects the key, run the agent with `-pair` again.

Location is fixed for now: add `-latitude 9.03 -longitude 38.74`. Without both, none is reported.

## 3. Send a command

```bash
curl -X POST http://localhost:8080/api/devices/linux-pc-01/command -H "X-OWNER-KEY: $(cat data/owner-key)" -H 'Content-Type: application/json' -d '{"type":"MESSAGE","payload":"Hello"}'
```

| Command | On the laptop |
|---|---|
| `RING` | Plays an alarm sound (`pw-play`, `paplay` or `canberra-gtk-play`) |
| `LOCK` | Locks the session (`loginctl`, else `xdg-screensaver`) |
| `MESSAGE` | Shows `payload` as a notification (`notify-send`) |

Commands for an offline device wait until it reconnects.

## Test

```bash
mvn test
```

```bash
go test ./...
```

## Settings

| Variable | Default | |
|---|---|---|
| `PORT` | `8080` | |
| `OWNER_API_KEY` | generated | Owner secret |
| `TRACKER_DATA_DIR` | `./data` | Absolute, or starting with `./` |
| `SPRING_PROFILES_ACTIVE` | | `prod` uses PostgreSQL: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` |

Before exposing it to the internet, use the `prod` profile behind HTTPS and run one
instance only (live connections are held in memory). `prod` trusts forwarded headers,
so it must only be reachable through the proxy.

## Not done yet

- Real location on Linux
- A phone app
- Rate limiting of wrong keys
# tracker
