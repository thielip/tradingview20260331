// This work is licensed under Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International 
// https://creativecommons.org/licenses/by-nc-sa/4.0/
// Optimized & Merged by Gemini
// © BigBeluga & Community

//@version=6
indicator("All-in-One: Trend, S/R, Dashboard & OB", overlay = true, max_lines_count = 500, max_labels_count = 500, max_boxes_count = 500)

// ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――
// 1. 參數設定 (INPUTS)
// ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――

// --- Group: Trend Filter (2-pole) ---
grp_tf = "趨勢濾波器 (Trend Filter)"
tf_length   = input.int(20, "Length", group = grp_tf)
tf_damping  = input.float(0.9, "Damping", minval = 0.1, maxval = 1.0, step = 0.01, group = grp_tf)
tf_ris_fal  = input.int(5, "Rising and Falling Threshold", group = grp_tf)
tf_bands    = input.float(1.0, "ATR Bands Multiplier", step = 0.1, minval = 0.5, group = grp_tf)
tf_up_col   = input.color(color.lime, "上漲顏色", inline = "tf_col", group = grp_tf)
tf_dn_col   = input.color(color.red, "下跌顏色", inline = "tf_col", group = grp_tf)
tf_neu_col  = input.color(color.yellow, "中性顏色", inline = "tf_col", group = grp_tf)
tf_bar_col  = input.bool(false, "K棒上色", group = grp_tf)
tf_signals  = input.bool(false, "顯示買賣訊號圖標", group = grp_tf)

// --- Group: Support & Resistance (S/R) ---
grp_sr = "支撐與壓力 (Support & Resistance)"
sr_prd          = input.int(10, 'Pivot Period', minval=4, maxval=30, group=grp_sr)
sr_src_type     = input.string('High/Low', 'Source Type', options=['High/Low', 'Close/Open'], group=grp_sr)
sr_maxnumpp     = input.int(20, 'Max Pivot Points', minval=5, maxval=100, group=grp_sr)
sr_channelW     = input.int(10, 'Max Channel Width %', minval=1, group=grp_sr)
sr_maxnumsr     = input.int(5, 'Max S/R Lines', minval=1, maxval=10, group=grp_sr)
sr_min_str      = input.int(2, 'Min Strength', minval=1, maxval=10, group=grp_sr)
sr_labelloc     = input.int(20, 'Label Offset', group=grp_sr)
sr_linestyle    = input.string('Dashed', 'Line Style', options=['Solid', 'Dotted', 'Dashed'], group=grp_sr)
sr_linewidth    = input.int(2, 'Line Width', minval=1, maxval=4, group=grp_sr)
sr_res_col      = input.color(color.red, 'Resistance Color', group=grp_sr)
sr_sup_col      = input.color(color.lime, 'Support Color', group=grp_sr)
sr_showpp       = input.bool(false, 'Show Pivot Points Labels', group=grp_sr)

// --- Group: Dashboard Indicators ---
grp_db = "資訊面板 (Dashboard)"
db_show         = input.bool(true, "顯示面板", group = grp_db)
db_rsiLength    = input.int(14, "RSI Length", group = grp_db)
db_kLength      = input.int(9,  "KD K Length", group = grp_db)
db_dLength      = input.int(3,  "KD D Length", group = grp_db)
db_smoothK      = input.int(3,  "KD Smooth K", group = grp_db)
db_mfiLength    = input.int(14, "MFI Length", group = grp_db)
db_macdFast     = input.int(12, "MACD Fast", group = grp_db)
db_macdSlow     = input.int(26, "MACD Slow", group = grp_db)
db_macdSignal   = input.int(9,  "MACD Signal", group = grp_db)
db_volAvgLen    = input.int(20, "Volume MA Length", group = grp_db)

// --- Group: Order Blocks (OB) ---
grp_ob = "訂單塊 (Order Blocks)"
ob_show         = input.bool(true, '顯示訂單塊', group = grp_ob)
ob_size         = input.int(5, '顯示數量', group = grp_ob)
ob_colBull      = input.color(color.new(#2157f3, 80), '看漲 OB', group = grp_ob)
ob_colBear      = input.color(color.new(#F23645, 80), '看跌 OB', group = grp_ob)

// ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――
// 2. 類型與函數定義 (TYPES & FUNCTIONS)
// ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――

// Type for Order Blocks
type orderBlock
    float barHigh
    float barLow
    int barTime
    int bias

// Function: Two-pole filter
method two_pole_filter(float src, int length, float damping) =>
    float omega = 2.0 * math.pi / length
    float alpha = damping * omega
    float beta = math.pow(omega, 2)
    var float f1 = na
    var float f2 = na
    f1 := nz(f1[1]) + alpha * (src - nz(f1[1]))
    f2 := nz(f2[1]) + beta * (f1 - nz(f2[1]))
    f2

// Function: Dashboard Row Filler
f_fillRow(_table, _r, _n, _v, _prevV, _advice, _c) =>
    table.cell(_table, 0, _r, _n, bgcolor=color.white, text_color=color.black)
    table.cell(_table, 1, _r, str.tostring(_v, "#.##"), bgcolor=color.white, text_color=color.black)
    table.cell(_table, 2, _r, _v > _prevV ? "🔺" : "🔻", text_color=_v > _prevV ? color.red : color.green, bgcolor=color.white)
    table.cell(_table, 3, _r, _advice, text_color=_c, bgcolor=color.white)

// Function: S/R Helper - Find location in strength array
find_loc(strength, _sr_strength) =>
    ret = array.size(_sr_strength)
    for i = ret > 0 ? array.size(_sr_strength) - 1 : na to 0 by 1
        if strength <= array.get(_sr_strength, i)
            break
        ret := i
    ret

// ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――
// 3. 邏輯計算 (CALCULATIONS)
// ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――

// === A. Trend Filter Calculation ===
tf_atr = ta.atr(200) * tf_bands
tp_f = close.two_pole_filter(tf_length, tf_damping)

var tf_rising = 0
var tf_falling = 0

tf_up = tp_f > tp_f[2]
tf_dn = tp_f < tp_f[2]

if tf_up
    tf_rising += 1
    tf_falling := 0
if tf_dn
    tf_rising := 0
    tf_falling += 1

tf_color = tf_up ? color.from_gradient(tf_rising, 0, 15, tf_neu_col, tf_up_col) : tf_dn ? color.from_gradient(tf_falling, 0, 15, tf_neu_col, tf_dn_col) : tf_neu_col

// === B. Support & Resistance Calculation ===
float sr_src1 = sr_src_type == 'High/Low' ? high : math.max(close, open)
float sr_src2 = sr_src_type == 'High/Low' ? low : math.min(close, open)
float ph_sr = ta.pivothigh(sr_src1, sr_prd, sr_prd)
float pl_sr = ta.pivotlow(sr_src2, sr_prd, sr_prd)

// S/R Global Variables
var pivotvals = array.new_float(0)
var sr_up_level = array.new_float(0)
var sr_dn_level = array.new_float(0)
var sr_strength = array.new_float(0)

// Pre-calculate style
Lstyle = sr_linestyle == 'Dashed' ? line.style_dashed : sr_linestyle == 'Solid' ? line.style_solid : line.style_dotted

// Calculate max channel width
prdhighest = ta.highest(300)
prdlowest = ta.lowest(300)
cwidth = (prdhighest - prdlowest) * sr_channelW / 100

// Update Pivot Array
if not na(ph_sr) or not na(pl_sr)
    array.unshift(pivotvals, not na(ph_sr) ? ph_sr : pl_sr)
    if array.size(pivotvals) > sr_maxnumpp 
        array.pop(pivotvals)

// S/R Logic Function (Embedded to access scope)
get_sr_vals(ind) =>
    float lo = array.get(pivotvals, ind)
    float hi = lo
    int numpp = 0
    for y = 0 to array.size(pivotvals) - 1 by 1
        float cpp = array.get(pivotvals, y)
        float wdth = cpp <= lo ? hi - cpp : cpp - lo
        if wdth <= cwidth 
            if cpp <= hi
                lo := math.min(lo, cpp)
            else
                hi := math.max(hi, cpp)
            numpp += 1
    [hi, lo, numpp]

check_sr(hi, lo, strength) =>
    ret = true
    for i = 0 to array.size(sr_up_level) > 0 ? array.size(sr_up_level) - 1 : na by 1
        if (array.get(sr_up_level, i) >= lo and array.get(sr_up_level, i) <= hi) or (array.get(sr_dn_level, i) >= lo and array.get(sr_dn_level, i) <= hi)
            if strength >= array.get(sr_strength, i)
                array.remove(sr_strength, i)
                array.remove(sr_up_level, i)
                array.remove(sr_dn_level, i)
                ret
            else
                ret := false
            break
    ret

// S/R Re-calculation on new pivot
if not na(ph_sr) or not na(pl_sr)
    array.clear(sr_up_level)
    array.clear(sr_dn_level)
    array.clear(sr_strength)
    for x = 0 to array.size(pivotvals) - 1 by 1
        [hi, lo, strength] = get_sr_vals(x)
        if check_sr(hi, lo, strength)
            loc = find_loc(strength, sr_strength)
            if loc < sr_maxnumsr and strength >= sr_min_str
                array.insert(sr_strength, loc, strength)
                array.insert(sr_up_level, loc, hi)
                array.insert(sr_dn_level, loc, lo)
                if array.size(sr_strength) > sr_maxnumsr
                    array.pop(sr_strength)
                    array.pop(sr_up_level)
                    array.pop(sr_dn_level)

// === C. Dashboard Indicator Calculation ===
rsiValue = ta.rsi(close, db_rsiLength)
lLow_kd  = ta.lowest(low, db_kLength)
hHigh_kd = ta.highest(high, db_kLength)
kRaw     = (hHigh_kd - lLow_kd) != 0 ? 100 * (close - lLow_kd) / (hHigh_kd - lLow_kd) : 0
kValue   = ta.sma(kRaw, db_smoothK)
dValue   = ta.sma(kValue, db_dLength)
mfiValue = ta.mfi(hlc3, db_mfiLength)
[macdLine, signalLine, hist] = ta.macd(close, db_macdFast, db_macdSlow, db_macdSignal)
volAvg   = ta.sma(volume, db_volAvgLen)

// === D. Order Block Calculation ===
var float ob_p_high_level = 0.0
var int ob_p_high_idx = 0
var float ob_p_low_level = 0.0
var int ob_p_low_idx = 0

var obs = array.new<orderBlock>()
var obBoxes = array.new<box>()

// OB Cache
var p_highs_val = array.new<float>()
var p_lows_val  = array.new<float>()
var p_times_val = array.new<int>()

array.push(p_highs_val, high)
array.push(p_lows_val, low)
array.push(p_times_val, time)

if array.size(p_highs_val) > 500
    array.shift(p_highs_val)
    array.shift(p_lows_val)
    array.shift(p_times_val)

if barstate.isfirst and ob_show
    for i = 1 to ob_size
        array.push(obBoxes, box.new(na, na, na, na, xloc = xloc.bar_time, extend = extend.right))

// Separate Pivots for OB (Hardcoded 5 in original, kept to maintain logic)
ph_ob = ta.pivothigh(high, 5, 5)
pl_ob = ta.pivotlow(low, 5, 5)

if not na(ph_ob)
    ob_p_high_level := ph_ob
    ob_p_high_idx   := bar_index[5]
if not na(pl_ob)
    ob_p_low_level  := pl_ob
    ob_p_low_idx    := bar_index[5]

// OB Creation
if ta.crossover(close, ob_p_high_level) and ob_p_high_idx > 0
    startIdx = bar_index - ob_p_high_idx
    if startIdx > 0 and array.size(p_lows_val) >= startIdx
        a_slice = array.slice(p_lows_val, array.size(p_lows_val) - startIdx)
        if array.size(a_slice) > 0
            m_val = array.min(a_slice)
            m_idx = array.indexof(a_slice, m_val)
            target_high = array.get(p_highs_val, array.size(p_highs_val) - array.size(a_slice) + m_idx)
            target_time = array.get(p_times_val, array.size(p_times_val) - array.size(a_slice) + m_idx)
            array.unshift(obs, orderBlock.new(target_high, m_val, target_time, 1))
            ob_p_high_level := na

if ta.crossunder(close, ob_p_low_level) and ob_p_low_idx > 0
    startIdx = bar_index - ob_p_low_idx
    if startIdx > 0 and array.size(p_highs_val) >= startIdx
        a_slice = array.slice(p_highs_val, array.size(p_highs_val) - startIdx)
        if array.size(a_slice) > 0
            m_val = array.max(a_slice)
            m_idx = array.indexof(a_slice, m_val)
            target_low = array.get(p_lows_val, array.size(p_lows_val) - array.size(a_slice) + m_idx)
            target_time = array.get(p_times_val, array.size(p_times_val) - array.size(a_slice) + m_idx)
            array.unshift(obs, orderBlock.new(m_val, target_low, target_time, -1))
            ob_p_low_level := na

// OB Mitigation
if array.size(obs) > 0
    for i = array.size(obs) - 1 to 0
        ob = array.get(obs, i)
        if (close > ob.barHigh and ob.bias == -1) or (close < ob.barLow and ob.bias == 1)
            array.remove(obs, i)

// ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――
// 4. 繪圖與視覺化 (PLOTTING & VISUALS)
// ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――

// === Draw Trend Filter ===
plot(tp_f, "Two-Pole Filter", color = tf_color, linewidth = 3)

plotshape(tf_falling >= tf_ris_fal ? tp_f + tf_atr : na, "Falling Trend", shape.circle, location.absolute, color = tf_color)
plotshape(tf_rising >= tf_ris_fal  ? tp_f - tf_atr : na, "Rising Trend", shape.circle, location.absolute, color = tf_color)

bool tf_sig_up = ta.crossover(tf_rising, tf_ris_fal) and barstate.isconfirmed and tf_signals
bool tf_sig_dn = ta.crossover(tf_falling, tf_ris_fal) and barstate.isconfirmed and tf_signals

plotshape(tf_sig_dn ? tp_f[1] + tf_atr : na, "Signal Down", shape.triangledown, location.absolute, color = tf_color, size = size.tiny, offset = -1)
plotshape(tf_sig_up ? tp_f[1] - tf_atr : na, "Signal Up", shape.triangleup, location.absolute, color = tf_color, size = size.tiny, offset = -1)

barcolor(tf_bar_col ? tf_color : na)

// === Draw S/R ===
plotshape(not na(ph_sr) and sr_showpp, text='H', style=shape.labeldown, color=na, textcolor=color.new(color.red, 0), location=location.abovebar, offset=-sr_prd)
plotshape(not na(pl_sr) and sr_showpp, text='L', style=shape.labelup, color=na, textcolor=color.new(color.lime, 0), location=location.belowbar, offset=-sr_prd)

// Manage S/R Lines and Labels (Dynamic)
var sr_lines = array.new_line(11, na)
var sr_labels = array.new_label(11, na)

// Clear old drawings each bar (standard method for dynamic S/R)
for x = 1 to 10 by 1
    line.delete(array.get(sr_lines, x))
    label.delete(array.get(sr_labels, x))

for x = 0 to array.size(sr_up_level) > 0 ? array.size(sr_up_level) - 1 : na by 1
    float mid = math.round_to_mintick((array.get(sr_up_level, x) + array.get(sr_dn_level, x)) / 2)
    rate = 100 * (mid - close) / close
    
    // Draw only valid lines
    array.set(sr_labels, x + 1, label.new(x=bar_index + sr_labelloc, y=mid, text=str.tostring(mid) + '(' + str.tostring(rate, '#.##') + '%)', color=mid >= close ? color.red : color.lime, textcolor=mid >= close ? color.white : color.black, style=mid >= close ? label.style_label_down : label.style_label_up))
    array.set(sr_lines, x + 1, line.new(x1=bar_index, y1=mid, x2=bar_index - 1, y2=mid, extend=extend.both, color=mid >= close ? sr_res_col : sr_sup_col, style=Lstyle, width=sr_linewidth))

// S/R Alerts
sr_crossed_over = false
sr_crossed_under = false
for x = 0 to array.size(sr_up_level) > 0 ? array.size(sr_up_level) - 1 : na by 1
    float mid = math.round_to_mintick((array.get(sr_up_level, x) + array.get(sr_dn_level, x)) / 2)
    if close[1] <= mid and close > mid
        sr_crossed_over := true
    if close[1] >= mid and close < mid
        sr_crossed_under := true

alertcondition(sr_crossed_over, title='Resistance Broken', message='Resistance Broken')
alertcondition(sr_crossed_under, title='Support Broken', message='Support Broken')

// === Draw Dashboard ===
var table myTable = table.new(position.top_right, 4, 7, border_width=1, border_color=color.black, frame_color=color.black, frame_width=1)

if barstate.islast and db_show
    isVolHigh    = volume > volAvg 
    volTextColor = isVolHigh ? color.red : color.green

    // Headers
    table.cell(myTable, 0, 0, "指標", bgcolor=color.white, text_color=color.black)
    table.cell(myTable, 1, 0, "數值", bgcolor=color.white, text_color=color.black)
    table.cell(myTable, 2, 0, "趨勢", bgcolor=color.white, text_color=color.black)
    table.cell(myTable, 3, 0, "買賣建議", bgcolor=color.white, text_color=color.black)

    // Volume
    table.cell(myTable, 0, 1, "Volume", bgcolor=color.white, text_color=color.black)
    table.cell(myTable, 1, 1, str.tostring(volume, format.volume), text_color=volTextColor, bgcolor=color.white)
    table.cell(myTable, 2, 1, isVolHigh ? "🔺爆量" : "🔻萎縮", text_color=volTextColor, bgcolor=color.white)
    table.cell(myTable, 3, 1, isVolHigh ? "放量 (高於均線)" : "縮量 (低於均線)", text_color=volTextColor, bgcolor=color.white)

    // Rows
    f_fillRow(myTable, 2, "RSI", rsiValue, rsiValue[1], (rsiValue > 70 ? "超買(警惕)" : rsiValue < 30 ? "超跌(關注)" : "走勢中性"), (rsiValue > 70 ? color.red : rsiValue < 30 ? color.green : color.black))
    f_fillRow(myTable, 3, "KD-K", kValue, kValue[1], (kValue > 80 ? "多方強勢" : kValue < 20 ? "空方強勢" : "動能偏弱"), (kValue > 80 ? color.red : kValue < 20 ? color.green : color.black))
    f_fillRow(myTable, 4, "KD-D", dValue, dValue[1], (dValue > 80 ? "高檔鈍化" : dValue < 20 ? "低檔盤整" : "趨勢維持"), (dValue > 80 ? color.red : dValue < 20 ? color.green : color.black))
    f_fillRow(myTable, 5, "MFI", mfiValue, mfiValue[1], (mfiValue > 80 ? "資金過熱" : mfiValue < 20 ? "資金枯竭" : "流量穩定"), (mfiValue > 80 ? color.red : mfiValue < 20 ? color.green : color.black))
    
    // MACD
    table.cell(myTable, 0, 6, "MACD", bgcolor=color.white, text_color=color.black)
    table.cell(myTable, 1, 6, str.tostring(macdLine, "#.####"), bgcolor=color.white, text_color=color.black)
    table.cell(myTable, 2, 6, macdLine > macdLine[1] ? "🔺" : "🔻", text_color=macdLine > 0 ? color.red : color.green, bgcolor=color.white)
    table.cell(myTable, 3, 6, macdLine > 0 ? "零軸上(偏多)" : "零軸下(偏空)", text_color=macdLine > 0 ? color.red : color.green, bgcolor=color.white)

// === Draw Order Blocks ===
if ob_show and barstate.islast
    for b in obBoxes
        box.set_lefttop(b, na, na)
        box.set_rightbottom(b, na, na)
    if array.size(obs) > 0
        for i = 0 to math.min(array.size(obs), ob_size) - 1
            o = array.get(obs, i)
            bx = array.get(obBoxes, i)
            bCol = o.bias == 1 ? ob_colBull : ob_colBear
            box.set_lefttop(bx, o.barTime, o.barHigh)
            box.set_rightbottom(bx, o.barTime, o.barLow)
            box.set_bgcolor(bx, bCol)
            box.set_border_color(bx, bCol)


