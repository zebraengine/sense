# sense_api
## Sense Energy Monitor API Interface

The Sense API provides access to the unofficial API for the Sense Energy Monitor. Through the API,
one can retrieve both realtime and trend data, including individual devices, solar production, and
grid import/export.

Systematic access to the Sense monitor data. Exploratory work on pulling data from Sense
to be used in other tools - Home Assistant, SmartThings, ActionTiles, etc.

Python version based on the work done here in Powershell:
https://gist.github.com/mbrownnycnyc/db3209a1045746f5e287ea6b6631e19c

### Install

```
pip install sense_energy
```

## Package Contents

The `sense_energy` package exposes:

| Class | Purpose |
|-------|---------|
| `Senseable` | Synchronous API client (`requests` + `websocket-client`) |
| `ASyncSenseable` | Asynchronous API client (`aiohttp` + `websockets`) — recommended for integrations like Home Assistant |
| `SenseLink` | Local emulation of TP-Link Kasa HS110 smart plugs, to report custom power usage *to* your Sense monitor |
| `PlugInstance` | A single emulated plug used with `SenseLink` |
| `SenseDevice` | Model for a discovered device (`id`, `name`, `icon`, `is_on`, `power_w`, `energy_kwh[scale]`) |
| `Scale` | Enum of trend scales: `DAY`, `WEEK`, `MONTH`, `YEAR`, `CYCLE` (billing cycle) |

Both clients share the same base class (`SenseableBase`) and expose a nearly identical set of
methods and properties — the async client simply requires `await`.

## Quick Start (synchronous)

```python
    from sense_energy import Senseable
    sense = Senseable()
    sense.authenticate(username, password)
    sense.update_realtime()
    sense.update_trend_data()
    print ("Active:", sense.active_power, "W")
    print ("Active Solar:", sense.active_solar_power, "W")
    print ("Daily:", sense.daily_usage, "KWh")
    print ("Daily Solar:", sense.daily_production, "KWh")
    print ("Active Devices:", ", ".join(sense.active_devices))
```

## Quick Start (asynchronous)

```python
    import asyncio
    from sense_energy import ASyncSenseable

    async def main():
        sense = ASyncSenseable()
        await sense.authenticate(username, password)
        await sense.update_realtime()
        await sense.update_trend_data()
        print("Active:", sense.active_power, "W")
        print("Daily:", sense.daily_usage, "KWh")
        for device in sense.devices:
            print(device.name, device.power_w, "W", "on" if device.is_on else "off")

    asyncio.run(main())
```

## Authentication

### Username / password

```python
    sense.authenticate(username, password)
```

If the account has multi-factor authentication enabled, `authenticate()` raises
`SenseMFARequiredException`. Complete the login by supplying the TOTP code:

```python
    from sense_energy import Senseable, SenseMFARequiredException

    sense = Senseable()
    try:
        sense.authenticate(username, password)
    except SenseMFARequiredException:
        code = input("Enter MFA code: ")
        sense.validate_mfa(code)
```

### Reusing a session (recommended)

Authenticating too frequently may get rate limited — you should authenticate at most once every
15–20 minutes, and ideally only once, persisting the tokens instead. After authenticating, save
`sense.sense_access_token`, `sense.sense_user_id`, `sense.device_id`, and `sense.refresh_token`,
then restore the session later without re-sending credentials:

```python
    sense = Senseable()
    sense.load_auth(access_token, user_id, device_id, refresh_token)
    sense.set_monitor_id(monitor_id)   # also save/restore your monitor id
    sense.update_realtime()
```

Expired access tokens are renewed automatically using the refresh token; you can also call
`renew_auth()` explicitly.

### TLS / certificate verification

All HTTPS and websocket connections verify the server certificate by default. Both clients accept
`ssl_verify=False` (disable verification — not recommended; exposes your access token to
interception) and `ssl_cafile="/path/to/ca.pem"` (verify against a custom CA bundle, e.g. behind a
corporate TLS proxy) as constructor arguments. Credentials are only ever sent to Sense's own API
(`api.sense.com` / `clientrt.sense.com`) and are never written to disk by this library.

## Reading Data

If using the API to log data, create one instance of `Senseable`/`ASyncSenseable` and reuse it to
get updated stats.

- `update_realtime()` refreshes the realtime state (current power, per-device wattage and on/off
  status). It is rate limited to one call per 60 seconds by default; change this by setting the
  `rate_limit` attribute (seconds) on the client instance.
- `update_trend_data()` refreshes trend stats for all scales (day/week/month/year/billing cycle),
  including solar/production data when the monitor has solar configured.

### Realtime properties

Available after `update_realtime()`:

| Property | Description |
|----------|-------------|
| `active_power` | Current total consumption (W) |
| `active_solar_power` | Current solar production (W) |
| `active_voltage` | Voltage per leg (list) |
| `active_frequency` | Grid frequency (Hz) |
| `active_devices` | Names of devices currently on |
| `devices` | List of `SenseDevice` objects (name, icon, current wattage, on/off, energy per scale) |

### Trend properties

Available after `update_trend_data()`, for each of `daily_`, `weekly_`, `monthly_`, `yearly_`:

| Property suffix | Description |
|-----------------|-------------|
| `usage` | Energy consumed (kWh) |
| `production` | Solar energy produced (kWh) |
| `production_pct` | Production as % of consumption |
| `net_production` | Production minus consumption (kWh) |
| `from_grid` | Energy imported from grid (kWh) |
| `to_grid` | Energy exported to grid (kWh) |
| `solar_powered` | % of consumption covered by solar |

For example: `sense.daily_usage`, `sense.monthly_production`, `sense.yearly_to_grid`.

For other scales/keys use `get_stat(Scale.CYCLE, "consumption")` or
`get_trend("DAY", "production")` directly. `trend_start(scale)` and `trend_update(scale)` return
timestamps of when the trend window began and was last updated. `time_zone` returns the monitor's
time zone string.

### Method reference

Methods are the same on both clients (`await` them on `ASyncSenseable`) unless noted:

| Method | Description |
|--------|-------------|
| `authenticate(username, password)` | Log in with email/password; raises `SenseMFARequiredException` if MFA is needed |
| `validate_mfa(code)` | Complete an MFA login with a TOTP code |
| `load_auth(access_token, user_id, device_id, refresh_token)` | Restore a previous session without re-authenticating |
| `renew_auth()` | Refresh the access token using the refresh token |
| `logout()` | End the session |
| `set_monitor_id(monitor_id)` | Select the monitor to query (set automatically on authenticate) |
| `update_realtime()` | Refresh realtime data (rate limited; see above) |
| `get_realtime_stream()` | *(sync only)* Generator yielding continuous realtime updates from the websocket |
| `async_realtime_stream(callback, single)` | *(async only)* Stream realtime updates from the websocket, optionally invoking a callback |
| `get_realtime_future(callback)` | *(async only)* Returns a future for the realtime stream |
| `get_realtime_update()` | Fetch a single realtime snapshot via the REST API (monitor firmware ≥ 1.64) |
| `update_trend_data(dt=None)` | Refresh trend data for all scales, optionally as of a given datetime |
| `get_trend_data(scale, dt=None)` | Refresh trend data for one `Scale` |
| `get_stat(scale, key)` / `get_trend(scale, key)` | Read a value from the fetched trend data |
| `get_monitor_data()` | Monitor overview (also detects whether solar is configured) |
| `get_monitor_info()` | Monitor status & device-detection progress |
| `get_sw_version()` | Monitor firmware version (cached, re-checked daily) |
| `fetch_devices()` | *(async only)* Refresh the device list, merging duplicate smart plugs |
| `get_discovered_device_names()` | List of discovered device names |
| `get_discovered_device_data()` | Raw discovered-device data |
| `get_device_info(device_id)` | *(sync only)* Details for a specific device |
| `always_on_info()` | *(sync only)* "Always On" device info |
| `get_all_usage_data(payload={"n_items": 30})` | *(sync only)* Timeline/usage events; payload supports `n_items`, `device_id`, `prior_to_item`, `rollup` |

### Exceptions

All importable from `sense_energy`:

| Exception | Raised when |
|-----------|-------------|
| `SenseAuthenticationException` | Login, token renewal, or an API call fails authentication |
| `SenseMFARequiredException` | The account requires an MFA code to finish logging in |
| `SenseAPITimeoutException` | An API call or websocket read times out |
| `SenseWebsocketException` | The realtime websocket returns an error |
| `SenseAPIException` | Other API errors |

## Local Device Emulation

The `SenseLink` class emulates the energy monitoring functionality of TP-Link Kasa HS110 Smart
Plugs and allows you to report "custom" power usage to your Sense Home Energy Monitor. This
requires enabling "TP-Link HS110/HS300 Smart Plug" in the Sense app.

Based off the work of https://github.com/cbpowell/SenseLink

### Local emulation Example Usage:
```python
	async def test():
		import time
		def test_devices():
			devices = [PlugInstance("lamp1", start_time=time()-20, alias="Lamp", power=10), 
					   PlugInstance("fan1", start_time=time()-300, alias="Fan", power=140)]
			for d in devices:
				yield d
		sl = SenseLink(test_devices)
		await sl.start()
		try:
			await asyncio.sleep(180)  # Serve for 3 minutes
		finally:
			await sl.stop()

	if __name__ == "__main__":
		asyncio.run(test())
```

### Contributors

Feel free to fork and PR! 

https://github.com/kbickar

### Todo

- Add POST/PUT where/if applicable
- CLI
- Improved error handling
- Tests
