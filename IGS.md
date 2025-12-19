 strategy("Autonomous Trading Agent Strategy",
    3          overlay=true,
    4          default_qty_type=strategy.percent_of_equity,
    5          default_qty_value=10,
    6          commission_type=strategy.commission.percent,
    7          commission_value=0.1)
    8 
    9 // ============================================================================
   10 // INPUTS - Strategy Parameters
   11 // ============================================================================
   12 
   13 // Moving Average Settings
   14 ma_fast_length = input.int(9, "Fast MA Length", minval=1, group="Moving Averages")
   15 ma_slow_length = input.int(21, "Slow MA Length", minval=1, group="Moving Averages")
   16 ma_type = input.string("EMA", "MA Type", options=["SMA", "EMA"], group="Moving Averages")
   17 
   18 // RSI Settings
   19 rsi_length = input.int(14, "RSI Length", minval=1, group="RSI")
   20 rsi_oversold = input.int(30, "RSI Oversold", minval=0, maxval=50, group="RSI")
   21 rsi_overbought = input.int(70, "RSI Overbought", minval=50, maxval=100, group="RSI")
   22 
   23 // Risk Management
   24 stop_loss_pct = input.float(2.0, "Stop Loss %", minval=0.1, maxval=10, step=0.1, group="Risk Management")
   25 take_profit_pct = input.float(4.0, "Take Profit %", minval=0.1, maxval=20, step=0.1, group="Risk Management")
   26 trailing_stop_pct = input.float(1.5, "Trailing Stop %", minval=0.1, maxval=10, step=0.1, group="Risk Management")
   27 
   28 // Webhook Settings
   29 webhook_url = input.string("", "Webhook URL (n8n)", group="Webhook")
   30 enable_webhook = input.bool(true, "Enable Webhook Alerts", group="Webhook")
   31 
   32 // Trading Hours
   33 use_trading_hours = input.bool(false, "Use Trading Hours", group="Trading Hours")
   34 trading_start_hour = input.int(9, "Start Hour", minval=0, maxval=23, group="Trading Hours")
   35 trading_end_hour = input.int(16, "End Hour", minval=0, maxval=23, group="Trading Hours")
   36 
   37 // ============================================================================
   38 // CALCULATIONS
   39 // ============================================================================
   40 
   41 // Moving Averages
   42 ma_fast = ma_type == "EMA" ? ta.ema(close, ma_fast_length) : ta.sma(close, ma_fast_length)
   43 ma_slow = ma_type == "EMA" ? ta.ema(close, ma_slow_length) : ta.sma(close, ma_slow_length)
   44 
   45 // RSI
   46 rsi = ta.rsi(close, rsi_length)
   47 
   48 // ATR for volatility
   49 atr = ta.atr(14)
   50 
   51 // Volume Analysis
   52 volume_avg = ta.sma(volume, 20)
   53 volume_spike = volume > volume_avg * 1.5
   54 
   55 // Trading Hours Check
   56 in_trading_hours = true
   57 if use_trading_hours
   58     current_hour = hour(time)
   59     in_trading_hours := current_hour >= trading_start_hour and current_hour < trading_end_hour
   60 
   61 // ============================================================================
   62 // STRATEGY LOGIC - Autonomous Agent Conditions
   63 // ============================================================================
   64 
   65 // LONG CONDITIONS
   66 long_condition_ma = ta.crossover(ma_fast, ma_slow)
   67 long_condition_rsi = rsi < rsi_oversold
   68 long_condition_volume = volume_spike
   69 long_signal = long_condition_ma and in_trading_hours
   70 
   71 // SHORT CONDITIONS
   72 short_condition_ma = ta.crossunder(ma_fast, ma_slow)
   73 short_condition_rsi = rsi > rsi_overbought
   74 short_condition_volume = volume_spike
   75 short_signal = short_condition_ma and in_trading_hours
   76 
   77 // EXIT CONDITIONS
   78 exit_long_rsi = rsi > rsi_overbought
   79 exit_short_rsi = rsi < rsi_oversold
   80 
   81 // ============================================================================
   82 // POSITION SIZING & RISK CALCULATION
   83 // ============================================================================
   84 
   85 // Calculate position size based on risk
   86 risk_per_trade = strategy.equity * (stop_loss_pct / 100)
   87 position_size = risk_per_trade / (close * (stop_loss_pct / 100))
   88 
   89 // Stop Loss and Take Profit Levels
   90 long_stop_loss = close * (1 - stop_loss_pct / 100)
   91 long_take_profit = close * (1 + take_profit_pct / 100)
   92 short_stop_loss = close * (1 + stop_loss_pct / 100)
   93 short_take_profit = close * (1 - take_profit_pct / 100)
   94 
   95 // ============================================================================
   96 // STRATEGY EXECUTION
   97 // ============================================================================
   98 
   99 if long_signal and strategy.position_size == 0
  100     strategy.entry("Long", strategy.long)
  101     strategy.exit("Long Exit", "Long",
  102                   stop=long_stop_loss,
  103                   limit=long_take_profit,
  104                   trail_price=close * (1 + trailing_stop_pct / 100),
  105                   trail_offset=close * (trailing_stop_pct / 100))
  106 
  107 if short_signal and strategy.position_size == 0
  108     strategy.entry("Short", strategy.short)
  109     strategy.exit("Short Exit", "Short",
  110                   stop=short_stop_loss,
  111                   limit=short_take_profit,
  112                   trail_price=close * (1 - trailing_stop_pct / 100),
  113                   trail_offset=close * (trailing_stop_pct / 100))
  114 
  115 // Force exit on opposite signal
  116 if exit_long_rsi and strategy.position_size > 0
  117     strategy.close("Long", comment="RSI Exit")
  118 
  119 if exit_short_rsi and strategy.position_size < 0
  120     strategy.close("Short", comment="RSI Exit")
  121 
  122 // ============================================================================
  123 // WEBHOOK ALERTS - Send to n8n Autonomous Agent
  124 // ============================================================================
  125 
  126 // Prepare webhook message with all trading data
  127 webhook_message = '{"strategy": "Autonomous Agent", ' +
  128                   '"action": "{{strategy.order.action}}", ' +
  129                   '"ticker": "{{ticker}}", ' +
  130                   '"price": "{{close}}", ' +
  131                   '"time": "{{time}}", ' +
  132                   '"position_size": "{{strategy.position_size}}", ' +
  133                   '"rsi": "' + str.tostring(rsi, "#.##") + '", ' +
  134                   '"ma_fast": "' + str.tostring(ma_fast, "#.##") + '", ' +
  135                   '"ma_slow": "' + str.tostring(ma_slow, "#.##") + '", ' +
  136                   '"atr": "' + str.tostring(atr, "#.##") + '", ' +
  137                   '"stop_loss": "' + str.tostring(long_stop_loss, "#.##") + '", ' +
  138                   '"take_profit": "' + str.tostring(long_take_profit, "#.##") + '", ' +
  139                   '"volume_spike": "' + str.tostring(volume_spike) + '"}'
  140 
  141 // Send alerts on trade execution
  142 if enable_webhook and webhook_url != ""
  143     if long_signal
  144         alert(webhook_message, alert.freq_once_per_bar_close)
  145     if short_signal
  146         alert(webhook_message, alert.freq_once_per_bar_close)
  147     if strategy.position_size[1] != 0 and strategy.position_size == 0
  148         alert('{"strategy": "Autonomous Agent", "action": "EXIT", "ticker": "{{ticker}}", "price": "{{close}}"}',
  149               alert.freq_once_per_bar_close)
  150 
  151 // ============================================================================
  152 // VISUALIZATION
  153 // ============================================================================
  154 
  155 // Plot Moving Averages
  156 plot(ma_fast, "Fast MA", color=color.new(color.blue, 0), linewidth=2)
  157 plot(ma_slow, "Slow MA", color=color.new(color.red, 0), linewidth=2)
  158 
  159 // Plot Entry/Exit Signals
  160 plotshape(long_signal, "Long Signal", shape.triangleup, location.belowbar,
  161           color=color.new(color.green, 0), size=size.small)
  162 plotshape(short_signal, "Short Signal", shape.triangledown, location.abovebar,
  163           color=color.new(color.red, 0), size=size.small)
  164 
  165 // Plot Stop Loss and Take Profit
  166 plot(strategy.position_size > 0 ? long_stop_loss : na, "Long SL",
  167      color=color.new(color.red, 0), style=plot.style_linebr, linewidth=1)
  168 plot(strategy.position_size > 0 ? long_take_profit : na, "Long TP",
  169      color=color.new(color.green, 0), style=plot.style_linebr, linewidth=1)
  170 plot(strategy.position_size < 0 ? short_stop_loss : na, "Short SL",
  171      color=color.new(color.red, 0), style=plot.style_linebr, linewidth=1)
  172 plot(strategy.position_size < 0 ? short_take_profit : na, "Short TP",
  173      color=color.new(color.green, 0), style=plot.style_linebr, linewidth=1)
  174 
  175 // Background color for trading hours
  176 bgcolor(in_trading_hours ? color.new(color.green, 95) : color.new(color.red, 95),
  177         title="Trading Hours")
  178 
  179 // ============================================================================
  180 // TABLE - Status Dashboard
  181 // ============================================================================
  182 
  183 var table status_table = table.new(position.top_right, 2, 8,
  184                                    border_width=1,
  185                                    border_color=color.gray,
  186                                    bgcolor=color.new(color.black, 85))
  187 
  188 if barstate.islast
  189     // Headers
  190     table.cell(status_table, 0, 0, "Metric", text_color=color.white, bgcolor=color.new(color.blue, 50))
  191     table.cell(status_table, 1, 0, "Value", text_color=color.white, bgcolor=color.new(color.blue, 50))
  192 
  193     // Data
  194     table.cell(status_table, 0, 1, "Position", text_color=color.white)
  195     table.cell(status_table, 1, 1, strategy.position_size > 0 ? "LONG" : strategy.position_size < 0 ? "SHORT" : "FLAT",
  196                text_color=strategy.position_size > 0 ? color.green : strategy.position_size < 0 ? color.red : color.gray)
  197 
  198     table.cell(status_table, 0, 2, "RSI", text_color=color.white)
  199     table.cell(status_table, 1, 2, str.tostring(rsi, "#.##"),
  200                text_color=rsi < rsi_oversold ? color.green : rsi > rsi_overbought ? color.red : color.white)
  201 
  202     table.cell(status_table, 0, 3, "Fast MA", text_color=color.white)
  203     table.cell(status_table, 1, 3, str.tostring(ma_fast, "#.##"), text_color=color.blue)
  204 
  205     table.cell(status_table, 0, 4, "Slow MA", text_color=color.white)
  206     table.cell(status_table, 1, 4, str.tostring(ma_slow, "#.##"), text_color=color.red)
  207 
  208     table.cell(status_table, 0, 5, "P&L", text_color=color.white)
  209     table.cell(status_table, 1, 5, str.tostring(strategy.netprofit, "#.##"),
  210                text_color=strategy.netprofit > 0 ? color.green : color.red)
  211 
  212     table.cell(status_table, 0, 6, "Win Rate", text_color=color.white)
  213     win_rate = strategy.wintrades / (strategy.wintrades + strategy.losstrades) * 100
  214     table.cell(status_table, 1, 6, str.tostring(win_rate, "#.##") + "%",
  215                text_color=win_rate > 50 ? color.green : color.red)
  216 
  217     table.cell(status_table, 0, 7, "Trading Hrs", text_color=color.white)
  218     table.cell(status_table, 1, 7, in_trading_hours ? "ACTIVE" : "CLOSED",
  219                text_color=in_trading_hours ? color.green : color.red)
