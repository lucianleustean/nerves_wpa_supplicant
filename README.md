# WpaSupplicant

[![Build Status](https://travis-ci.org/fhunleth/wpa_supplicant.ex.svg)](https://travis-ci.org/fhunleth/wpa_supplicant.ex)

This package enables Elixir applications to interact with the local WPA
supplicant. The WPA supplicant handles various Wi-Fi operations like scanning
for wireless networks, connecting, authenticating, and collecting wireless
adapter statistics.

## Building

The Makefile currently sets the `wpa_ex` binary to setuid root. This means that
`sudo` is used during the build process. To enter in your password during the
build, do the following:

    $ SUDO_ASKPASS=/usr/bin/ssh-askpass
    $ make

## Permissions

The `wpa_supplicant` process runs as root and requires processes that attach to
its control interface to be root. One way of doing this is to set the `wpa_ex`
binary to be setuid root. E.g.,

    chown root:root priv/wpa_ex
    chmod +s priv/wpa_ex

## Running

This code expects that the `wpa_supplicant` has already been started. If it
hasn't, start it. For example,

    iex> System.cmd "wpa_supplicant -iwlan0 -C/var/run/wpa_supplicant -B"

After the `wpa_supplicant` has been started, it's possible to start the
`WpaSupplicant` interface:

    iex> {:ok, pid} = WpaSupplicant.start_link("/var/run/wpa_supplicant/wlan0")
    {:ok, #PID<0.82.0}

    iex> WpaSupplicant.request(pid, :PING)
    :PONG

To scan for access points, call `WpaSupplicant.scan/1`. This can take a few
seconds:

    iex> WpaSupplicant.scan(pid)
    [%{age: 42, beacon_int: 100, bssid: "00:1f:90:db:45:54", capabilities: 1073,
       flags: "[WEP][ESS]", freq: 2462, id: 8,
       ie: "00053153555434010882848b0c1296182403010b07", 
       level: -83, noise: 0, qual: 0, ssid: "1SUT4", tsf: 580579066269},
     %{age: 109, beacon_int: 100, bssid: "00:18:39:7a:23:e8", capabilities: 1041,
       flags: "[WEP][ESS]", freq: 2412, id: 5,
       ie: "00076c696e6b737973010882848b962430486c0301",
       level: -86, noise: 0, qual: 0, ssid: "linksys", tsf: 464957892243},
     %{age: 42, beacon_int: 100, bssid: "1c:7e:e5:32:d1:f8", capabilities: 1041,
       flags: "[WPA2-PSK-CCMP][ESS]", freq: 2412, id: 0,
       ie: "000768756e6c657468010882848b960c1218240301",
       level: -43, noise: 0, qual: 0, ssid: "dlink", tsf: 580587711245}]

To attach to an access point, you need to configure a network entry in the
`wpa_supplicant`. The `wpa_supplicant` can have multiple network entries
configured. The following removes all network entries so that only one is
configured.

    iex> WpaSupplicant.request(pid, {:REMOVE_NETWORK, "all"})
    :ok
    # It's ok if :REMOVE_NETWORK fails. That just means there are no networks.

    iex> netid=WpaSupplicant.request(pid, :ADD_NETWORK)
    0

    iex> WpaSupplicant.request(pid, {:SET_NETWORK, netid, :ssid, "MyNetworkSsid"})
    :ok

    iex> WpaSupplicant.request(pid, {:SET_NETWORK, netid, :psk, "secret"})
    :ok

    # Once the network is enabled, the wpa_supplicant will start trying to
    # connect
    iex> WpaSupplicant.request(pid, {:ENABLE_NETWORK, netid})
    :ok

If the access point is around, the `wpa_supplicant` will eventually connect to
the network.

    iex> WpaSupplicant.request(pid, :STATUS)
    %{address: "84:3a:4b:11:95:23", bssid: "1c:7e:e5:32:de:32",
      group_cipher: "CCMP", id: 0, ip_address: "192.168.1.112",
      key_mgmt: "WPA2-PSK", mode: "station", pairwise_cipher: "CCMP",
      ssid: "MyNetworkSsid", uuid: "4724c801-d019-57b1-b58e-51f644312345",
      wpa_state: "COMPLETED"}

Polling the `wpa_supplicant` for status isn't that great, so it's possible to
register a `GenEvent` with `WpaSupplicant`. If you don't supply one, one is
automatically created and available via `WpaSupplicant.event_manager/1`. The
following example shows how to view events at the prompt:

    iex> defmodule Forwarder do
    ...>  use GenEvent
    ...>  def handle_event(event, parent) do
    ...>    send parent, event
    ...>    {:ok, parent}
    ...>  end
    ...> end
    iex> WpaSupplicant.event_manager(pid) |> GenEvent.add_handler(Forwarder, self())
    iex> flush
    {:wpa_supplicant, #PID<0.85.0>, :"CTRL-EVENT-SCAN-STARTED"}
    {:wpa_supplicant, #PID<0.85.0>, :"CTRL-EVENT-SCAN-RESULTS"}

## Useful links

  1. [wpa_supplicant homepage](http://w1.fi/wpa_supplicant/)
  2. [wpa_supplicant control
     interface](http://w1.fi/wpa_supplicant/devel/ctrl_iface_page.html)
  3. [wpa_supplicant information on the archlinux
     wiki](https://wiki.archlinux.org/index.php/Wpa_supplicant)

## Licensing

The majority of this package is licensed under the Apache 2.0 license. The code
that directly interfaces with the `wpa_supplicant` is taken from the
`wpa_supplicant` package and has the following copyright and license:

```
/*
 * wpa_supplicant/hostapd control interface library
 * Copyright (c) 2004-2007, Jouni Malinen <j@w1.fi>
 *
 * This software may be distributed under the terms of the BSD license.
 * See README for more details.
 */
```
