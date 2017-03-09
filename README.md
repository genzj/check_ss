# Shadowsocks Server Check for Nagios/Icinga

## What is it

This is a plugin for Nagios / Icinga to monitor the shadowsocks server status.

## What it does

This plugin starts a shadowsocks client in background then tries to fetch a web page over the proxy.

## How it works

This plugin issues [Shadowsocks-GO](https://github.com/shadowsocks/shadowsocks-go) client (because it's a single file distribution so easy to install) with given Shadowsocks parameters (server address, port, password, etc.) in the background.  Afterwards, with curl `--socks5-hostname` option, a HTTP request is done over shadowsocks client SOCKS interface.

## How to install
### Step 1: Get the script

__Warning: This instructions only apply to Icinga 2__

Fetch the script and place it in your `PluginDir`-Folder. That's usually `/usr/lib/nagios/plugins`.
If you are not sure about the value of `PluginDir` you can check `/etc/icinga2/constants.conf`.

### Step 2: Install Shadowsocks-go client

Download the proper **local** binary of [shadowsocks-go](https://github.com/shadowsocks/shadowsocks-go/releases) for your platform (tested with 1.1.5). Put it under any path accessible to Icinga 2 check then mark it executable. Here I assume it's downloaded as `/usr/bin/shadowsocks-local-linux64-1.1.5`.

### Step 3: Make it available for Icinga 2

#### Step 3.1: Create a CheckCommand object

Navigate on your Icinga server to your config folder, usually `/etc/icinga2/conf.d` and open the `commands.conf` file to add a new CheckCommand.

```
object CheckCommand "shadowsocks" {
  import "plugin-check-command"

  command = [
    PluginDir + "/check_ss",
    "--ss-path", "/usr/bin/shadowsocks-local-linux64-1.1.5",
  ] //constants.conf -> const PluginDir
  arguments = {
    "-A" = {
      set_if = "$ss_ota$"
    }
    "-b" = "$ss_local_addr$"
    "-k" = {
      required = true,
      value = "$ss_password$"
    }
    "-l" = {
      required = true,
      value = "$ss_local_port$"
    }
    "-m" = {
      required = true,
      value = "$ss_method$"
    }
    "-p" = {
      required = true,
      value = "$ss_server_port$"
    }
    "-s" = {
      required = true,
      value = "$ss_server_addr$"
    }
    "--max-time" = "$ss_max_time$"
    "--ss-path" = {
      required = true,
      value = "$ss_client_path$"
    }
    "--target" = "$ss_target$"
  }
  vars.ss_ota = false
  vars.ss_local_addr = "127.0.0.1"
  vars.ss_target = "https://www.google.com"
  vars.ss_max_time = 3000
  vars.ss_client_path = "/usr/bin/shadowsocks-local-linux64-1.1.5"
}
```

#### Step 3.2: Add check to a host

To add the check to a host use this snippet in a host and a service configuration file respectively:

```
object Host "proxy" {
  vars.ss["ss-1234"] = {
    ss_password = "password"
    ss_local_port = 10800
    ss_method = "salsa20"
    ss_server_port = 1234
  }
  vars.ss["ss-2345"] = {
    ss_password = "password"
    /* use different local port in case several checks are running
     * simultaneously and binding port collision leads to fake alarms. */
    ss_local_port = 10801
    ss_method = "salsa20"
    ss_server_port = 2345
  }
}
```

```
apply Service for (ss_service => config in host.vars.ss) {
  import "generic-service"

  check_command = "shadowsocks"

  vars.ss_server_addr = host.address
  vars += config
}
```

## TODO list

* also check proxy latency
* finish help message
* support log verbosity settings
