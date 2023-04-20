![image](https://user-images.githubusercontent.com/10352360/233438084-2bcc4734-9a6a-43cb-a728-e5152f3688c9.png)


Clock configuration with max7219digit and max7219 7-segment SPI displays with brightness correction during night time.


### Night mode
Night mode is used to reduce brightness during night time. It based on Home Assistant's scheduler(Settings - Devices and services - Helpers tab)
``` yaml
binary_sensor:
  - platform: homeassistant
    id: clock_night_mode
    entity_id: schedule.clock_night_mode
```

### Main display
The main MAX7219 4-matrix display is used to display the time from SNTP server. It uses pixel font `basis33`. If you are using ESPHome dashboard as a Home Assistant addon, place it to the `config\esphome\fonts` of Home Assistant. More info in [ESPHome docs](https://esphome.io/components/display/index.html?highlight=font#fonts)
```yaml
font:
  - file: "fonts/basis33.ttf"
    id: digit_font
    size: 16
    glyphs: "0123456789 "
  - file: "fonts/basis33.ttf"
    id: digit_font15
    size: 15
    glyphs: ": ."
    
display:
  - platform: max7219digit
    cs_pin: GPIO25
    num_chips: 4
    intensity: 0
    rotate_chip: 180
    reverse_enable: true
    lambda: |-
          if(!id(time_sync_done))
          {
            it.print(0, 5, id(digit_font15), ". . . . . . .");
            it.invert_on_off(true);
            return;
          }
          it.invert_on_off(false);
          auto time = id(sntp_time).now();
          bool is_night_mode = id(clock_night_mode).state;
          static int clock_should_blink = true;
          it.intensity(is_night_mode ? 0 : 5);
              
          it.strftime(0, 5, id(digit_font), TextAlign::CENTER_LEFT, "%H", time);
          
          if(is_night_mode || clock_should_blink)
          {
              it.strftime(12, 4, id(digit_font15), TextAlign::CENTER_LEFT, ":", time); 
          }
          it.strftime(17, 5, id(digit_font), TextAlign::CENTER_LEFT, "%M", time);
          clock_should_blink = !clock_should_blink;
          
```

### Secondary display
7-segment display is used to display indoor/outdoor temperatures from Home Assistant sensors

```yaml
  - platform: max7219
    cs_pin: GPIO4
    num_chips: 5 # 5 because it is on the same SPI bus with max7219digit
    intensity: 0
    update_interval: 1.5s
    lambda: |-
      static bool temp_outside = true;
      static int temp_mode_time = 0;
      auto sens = temp_outside ? id(outside_temp) : id(cgdk2_temperature);
      char text[7] = " --";

      if(!isnan(sens->state))
      {
        sprintf(text, "%5.1f", sens->state);
      }

      it.set_intensity(id(clock_night_mode).state ? 0 : 5);
      if(temp_outside)
        it.printf(0, "  o%2s", text); //#show outside temp
      else
        it.printf(0, "  h%2s", text); //#show room temp
        
      if(temp_mode_time > 2)//#switch to inside/outside sensor
      {
        temp_outside = !temp_outside;
        temp_mode_time = 0;
      }
      temp_mode_time++;
      
```
