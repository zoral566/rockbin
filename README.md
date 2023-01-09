

# rockbin



[![building](https://github.com/johnDorian/rockbin/actions/workflows/ci.yml/badge.svg)]((https://github.com/johnDorian/rockbin/actions/workflows/ci.yml/badge.svg))
[![codecov](https://codecov.io/gh/johnDorian/rockbin/branch/master/graph/badge.svg)](https://codecov.io/gh/johnDorian/rockbin)
[![gosec](https://goreportcard.com/badge/github.com/johnDorian/rockbin)]((https://goreportcard.com/badge/github.com/johnDorian/rockbin))
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2FjohnDorian%2Frockbin.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2FjohnDorian%2Frockbin?ref=badge_shield)

This repo contains the code for a simple go based mqtt client which will send the bin status to a mqtt server. 

I'm using home assistant, hence the home assistant auto discovery stuff.

The general idea is that the home assistant config is sent every minute, and the bin value is sent when the `/mnt/data/rockrobo/RoboController.cfg` is modified. The file is only modified after cleaning or returning to the dock. Bin value in RoboController.cfg is named: bin_in_time. This value is expressed in seconds from last cleaning of vacuum bin. After cleaning value is set to 0.

I decided to use a percentage value, in order to use a gauge in home assistant. I'm using 40 minutes as the time that the vacuum needs to be emptied. This can be changed as an input value to the mqtt client. 
It's also possible to create sensor expressed in minutes - please refer to parameters table.


## Contributing

Please contact me first with any suggestied changes, or before opening a PR. The main aim and scope of this project is to get the robot to report the bin status, so please keep this in mind when creating a PR. 


## Installation and setup.

The client can be started/tested using the following commands.

```bash
# copy the binary to the vacuum
scp rockbin root@...:/root/rockbin
# ssh into the vacuum
ssh root@...
# move the binary into the correct location
mv rockbin /usr/local/bin/rockbin
# make the binary executable
chmod +x rockbin
# setup the config file and install the required service script (e.g. /etc/init/S12rockbin). 
# This will overwrite any existing service scripts - please make a backup beforehand. 
/usr/local/bin/rockbin configure
# test the new version
/usr/local/bin/rockbin serve --/usr/local/bin/rockbin-daemon.sh debug
# If everything seems to be working finish restart the vacuum
reboot now
```
create /usr/local/bin/rockbin-daemon.sh
```bash
#!/bin/sh
#
# created by:  Atti


mkdir -p /var/log/upstart

while :; do
    sleep 5
    if [ `cut -d. -f1 /proc/uptime` -lt 300 ]; then
        echo -n "Waiting for 20 sec after boot..."
        sleep 20
        echo " done."
    fi

    pidof SysUpdate > /dev/null  2>&1
    if [ $? -ne 0 ]; then
    echo "Running Rockbin"
    /usr/local/bin/rockbin serve --config /mnt/data/rockbin/rockbin.yaml            
    else
    echo "Waiting for SysUpdate to finish..."
    fi
done
```
####################
```bash
cat /mnt/data/rockbin/rockbin.yaml
 
file_path: "/mnt/data/rockrobo/RoboController.cfg" 
full_time: "2400" 
log_level: "error" 
measurement_unit: "min" 
mqtt_password: "" 
mqtt_server: "mqtt://192.168.1.140:1883" 
mqtt_state_topic: "homeassistant/sensor/%v/state" 
mqtt_timeout: "5" 
mqtt_user: "" 
sensor_name: "vacuumbin" 
status_address: "0.0.0.0" 
status_port: "9999"
The above command set will setup the rockbin service and setup the configuration file according to your responses.
```
## Home assistant 
An example of sending the vacuum to the rubbish bin is below: 

```yaml
alias: Svetlana Send vacuum to the bin
description: ""
trigger:
  - platform: state
    entity_id:
      - vacuum.valetudo_svetlana
    to: docked
    for:
      hours: 0
      minutes: 0
      seconds: 3
    from: returning
condition:
  - condition: numeric_state
    entity_id: sensor.vacuumbin
    above: 100
action:
  - service: mqtt.publish
    data:
      topic: Homeassistant_svetlana/Svetlana/capabilities/BasicControlCapability
      payload: "{ \"action\": \"pause\" }"
    enabled: true
  - wait_template: "{{ is_state('vacuum.valetudo_svetlana', 'idle') }}"
    continue_on_timeout: "true"
    timeout: "00:00:05"
    enabled: true
  - service: mqtt.publish
    data:
      topic: Homeassistant_svetlana/Svetlana/GoToLocationCapability/go/set
      payload: "{   \"coordinates\": {     \"x\": 2511,     \"y\": 2503   } }"
  - service: telegram_bot.send_message
    data:
      message: Ürítsd ki a porszívó tartályát.
        
```
```yaml
alias: Svetlana Go home when bin is empty
description: ""
trigger:
  - platform: numeric_state
    entity_id: sensor.vacuumbin
    below: 1
action:
  - service: vacuum.return_to_base
    data: {}
    target:
      entity_id: vacuum.valetudo_svetlana
  - service: input_datetime.set_datetime
    data_template:
      datetime: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"
    target:
      entity_id: input_datetime.input_datetime_svetlana_last_emptied

```
Send message after cleaning
```yaml
alias: Svetlana porszivozas vege uzenet ha legalabb 5 percet takaritott
description: ""
trigger:
  - platform: state
    entity_id:
      - vacuum.valetudo_svetlana
    from: returning
    to: docked
condition:
  - condition: template
    value_template: >-
      value_template: '{{ ( trigger.to_state.attributes.sensor.vacuumbin|int -
      trigger.from_state.attributes.sensor.vacuumbin|int ) >= 5 }}' 
    enabled: false
action:
  - service: notify.telegram
    data:
      message: >-
        A porszívózás befejeződött. Feltakarítva: {{
        states("sensor.valetudo_svetlana_current_statistics_area") | int / 10000
        }} m2 , {{ states("sensor.valetudo_svetlana_current_statistics_time") |
        int / 60 }} perc alatt
initial_state: true
```
