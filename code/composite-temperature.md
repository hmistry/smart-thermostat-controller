# Composite Temperature Code

```jinja
{# Get room temperatures or set to default if none. Use a default within target window. #}
{% set t1 = states('sensor.atc_1_bh_temperature') | float(72.0) %}
{% set t2 = states('sensor.atc_2_bg_temperature') | float(72.0) %}
{% set t3 = states('sensor.atc_3_bm_temperature') | float(72.0) %}

{# Use room 1 & 3, ignore room 2 #}
{% set tmin = [ t1, t3 ] | min %}
{% set tmax = [ t1, t3 ] | max %}
{% set trange = 3.0 * 0.5 %}

{# Auto-select heating or cooling mode based on min temperature #}
{% if tmin < 75.0 %}
  {# heater mode #}
  
  {# Set target temperature based on thermostat preset mode as workaround for thermostat bug. #}
  {% if (state_attr('climate.thermostat_heater_upstairs', 'preset_mode') == "comfort") %}
    {% set tset =  71.5 %}
  {%- else -%}
    {% set tset = 70.5 %}
  {%- endif -%}

  {# heat if tmin < tset and tmax > max_temp_limit up to 75.0 #}
  {% if (tmax > (tset + trange) and tmin > tset) or (tmax > 75.0) %}
    {{ tmax }}
  {% else %}
    {{ tmin }}
  {% endif %}
  
{% else %}
  {# ac mode #}
  
  {# Set target temperature based on thermostat preset mode as workaround for thermostat bug. #}
  {% if (state_attr('climate.thermostat_ac_upstairs', 'preset_mode') == "comfort") %}
    {% set tset =  82.0 %}
  {%- else -%}
    {% set tset = 84.0 %}
  {%- endif -%}

  {# cool if tmax > tset and tmin > min_temp_limit down to 80.0 or if tdelta > 4.0 and tmax > 80.0 #} 
  {# tdelta is used to ignore temperature spikes due to direct sun heating sensor #}
  {% if ((tmax - tmin) > 4.0 and tmax > 80.0) or (tmin < (tset - trange) and tmax < tset) or (tmin < 80.0) %}
    {{ tmin }}
  {% else %}
    {{ tmax }}
  {% endif %}
{% endif %}
```
