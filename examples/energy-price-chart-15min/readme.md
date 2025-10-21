
<img width="611" height="350" alt="image" src="https://github.com/user-attachments/assets/356a8d5d-d748-42bb-8316-ac19010ada90" />



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
