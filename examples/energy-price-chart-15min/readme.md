
<img width="612" height="384" alt="image" src="https://github.com/user-attachments/assets/05d23c7c-29e7-4490-bb08-6385aec1195c" />


```YAML
type: custom:config-template-card
variables:
  PRICEHIGH: states['sensor.electricity_today_32nd_highest_price'].state
  PRICELOW: states['sensor.electricity_today_32nd_lowest_price'].state
entities:
  - sensor.electricity_prices
card:
  type: custom:apexcharts-card
  graph_span: 1d
  span:
    start: day
  apex_config:
    stroke:
      dashArray: 4
    chart:
      height: 250px
      width: 115%
      offsetX: -30
    title:
      text: Energy Price Today
      align: center
      offsetY: 0
      margin: 50
      style:
        fontSize: 13px
        fontFamily: Verdana
        fontWeight: normal
    grid:
      show: true
      borderColor: rgba(255,255,255,0.2)
    xaxis:
      position: bottom
      labels:
        format: H
        hideOverlappingLabels: true
        offsetX: 0
      axisTicks:
        offsetX: 0
      all_series_config:
        show:
          offset_in_name: true
    legend:
      show: false
      position: bottom
      horizontalAlign: left
      fontSize: 14px
      itemMargin:
        vertical: 10
        horizontal: 10
    tooltip:
      enabled: false
      style:
        fontSize: 14px
  header:
    title: Electricity Today
    standard_format: false
    show: false
    show_states: true
    colorize_states: true
  show:
    last_updated: true
  experimental:
    color_threshold: true
  now:
    show: true
  yaxis:
    - id: cost
      opposite: true
      decimals: 1
      apex_config:
        tickAmount: 4
        labels:
          show: true
        title:
          text: c/kWh
          rotate: 0
          offsetX: -25
          offsetY: -90
          style:
            fontSize: 10px
            fontFamily: verdana
            color: orange
    - id: energy
      max: ~2
      min: 0
      decimals: 1
      apex_config:
        tickAmount: 4
        labels:
          show: true
        title:
          text: kWh
          rotate: 0
          offsetX: 25
          offsetY: -90
          style:
            color: skyblue
            fontSize: 10px
            fontFamily: verdana
  series:
    - entity: sensor.electricity_prices
      name: Current Price
      yaxis_id: cost
      type: column
      opacity: 0.8
      stroke_width: 0
      show:
        legend_value: false
        extremas: true
        in_header: true
        header_color_threshold: true
      data_generator: |
        return entity.attributes.data.map(entry => {
          return [new Date(entry.start).getTime(), entry.price];
        });
      color_threshold:
        - value: -10
          color: lightgreen
        - value: ${PRICELOW * 1}
          color: orange
        - value: ${PRICEHIGH * 1}
          color: darkred
    - entity: sensor.electricity_daily_average_cents
      name: Average Price
      yaxis_id: cost
      type: line
      color: yellow
      stroke_width: 1
      opacity: 0.8
      group_by:
        func: last
        duration: 24h
      show:
        legend_value: false
        datalabels: false
        extremas: true
        in_header: true
    - entity: sensor.home_total_energy_hourly
      name: Energy Usage (kWh)
      color: skyblue
      type: line
      opacity: 1
      yaxis_id: energy
      stroke_width: 2
      float_precision: 1
      unit: kWh
      group_by:
        duration: 1h
        func: delta
      show:
        legend_value: false
        datalabels: false
        extremas: true
        in_header: raw
        header_color_threshold: true
```


```YAML
template:
  - triggers:
      - platform: time_pattern
        minutes: "/1"
      - platform: homeassistant
        event: start

    actions:
      - alias: "Fetch Today's Prices"
        service: nordpool.get_prices_for_date
        data:
          config_entry: XXXXXXXXXXXXXXXXXXXXXXX
          date: "{{ now().date() }}"
          areas: FI
          currency: EUR
        response_variable: today_price

      - alias: "Fetch Tomorrow's Prices"
        service: nordpool.get_prices_for_date
        data:
          config_entry: XXXXXXXXXXXXXXXXXXXXXXX
          date: "{{ now().date() + timedelta(days=1) }}"
          areas: FI
          currency: EUR
        response_variable: tomorrow_price

    sensor:
      - name: Electricity prices
        unique_id: electricity_prices
        unit_of_measurement: "c/kWh"
        icon: mdi:cash
        state: >
          {%- set region = (this.attributes.get('region', 'FI') | string) -%}
          {%- set tax = (this.attributes.get('tax', 1.0) | float) -%}
          {%- set additional_cost = (this.attributes.get('additional_cost', 0.0) | float) -%}

          {% if (today_price is mapping) and (tomorrow_price is mapping) %}
            {% set data = namespace(prices=[]) %}
            {% for state in today_price[region] %}
              {% set data.prices = data.prices + [(((state.price/10 | float)  * tax + additional_cost)) | round(3, default=0)] %}
            {% endfor %}
            {% for state in tomorrow_price[region] %}
              {% set data.prices = data.prices + [(((state.price/10 | float) * tax + additional_cost)) | round(3, default=0)] %}
            {% endfor %}
            {{min(data.prices)}}
          {% else %}
            unavailable
          {% endif %}
        attributes:
          tomorrow_valid: >
            {%- set region = (this.attributes.get('region', 'FI') | string) -%}
            {%- if (tomorrow_price is mapping) %}
              {%- if tomorrow_price[region] | list | count > 0 -%}
                {{ true | bool }}
              {%- else %}
                {{ false | bool }}
              {%- endif %}
            {%- else %}
              {{ false | bool }}
            {%- endif %}
          data: >
            {%- set region = (this.attributes.get('region', 'FI') | string) -%}
            {%- set tax = (this.attributes.get('tax', 1.0) | float) -%}
            {%- set additional_cost = (this.attributes.get('additional_cost', 0.0) | float) -%}

            {% if (today_price is mapping) and (tomorrow_price is mapping) %}
            {% set data = namespace(prices=[]) %}
              {% for state in today_price[region] %}
                {% set local_start = as_datetime(state.start).astimezone().strftime('%Y-%m-%d %H:%M:%S') %}
                {% set local_end = as_datetime(state.end).astimezone().strftime('%Y-%m-%d %H:%M:%S') %}
                {% set data.prices = data.prices + [{'start':local_start, 'end':local_end, 'price': (((state.price/10 | float) * tax + additional_cost)) | round(3, default=0)}] %}
              {% endfor %}
              {% for state in tomorrow_price[region] %}
                {% set local_start = as_datetime(state.start).astimezone().strftime('%Y-%m-%d %H:%M:%S') %}
                {% set local_end = as_datetime(state.end).astimezone().strftime('%Y-%m-%d %H:%M:%S') %}
                {% set data.prices = data.prices + [{'start':local_start, 'end':local_end, 'price': (((state.price/10 | float) * tax + additional_cost)) | round(3, default=0)}] %}
              {% endfor %}
              {{data.prices}}
            {% else %}
              []
            {% endif %}
          tax: "1"
          additional_cost: "0"
          region: FI

  - sensor:
      - name: "Electricity Daily Average (cents)"
        unique_id: electricity_daily_average_cents
        unit_of_measurement: "c/kWh"
        state_class: measurement
        state: >
          {{ (states('sensor.nord_pool_fi_daily_average') | float(0) * 100) | round(1) }}

  - sensor:
      - name: "Electricity Today 32nd Lowest Price"
        unique_id: electricity_today_32nd_lowest
        unit_of_measurement: "c/kWh"
        state: >
          {% set today = now().date().isoformat() %}
          {% set entries = state_attr('sensor.electricity_prices', 'data') %}
          {% if entries %}
            {% set today_prices = entries
              | selectattr('start', 'defined')
              | selectattr('start', 'string')
              | selectattr('start', 'search', today)
              | map(attribute='price')
              | list %}
            {% if today_prices | count >= 32 %}
              {% set sorted = today_prices | sort %}
              {{ sorted[31] | round(2) }}
            {% else %}
              none
            {% endif %}
          {% else %}
            none
          {% endif %}

  - sensor:
      - name: "Electricity Today 32nd Highest Price"
        unique_id: electricity_today_32nd_highest
        unit_of_measurement: "c/kWh"
        state: >
          {% set today = now().date() %}
          {% set entries = state_attr('sensor.electricity_prices', 'data') %}
          {% if entries %}
            {% set today_prices = entries
              | selectattr('start', 'defined')
              | selectattr('start', 'string')
              | selectattr('start', 'search', today.isoformat())
              | map(attribute='price')
              | list %}
            {% if today_prices | count >= 32 %}
              {% set sorted = today_prices | sort(reverse=true) %}
              {{ sorted[31] | round(2) }}
            {% else %}
              none
            {% endif %}
          {% else %}
            none
          {% endif %}
