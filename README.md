[Project] BABA MT5 Trading Bot — continue

[Platform] MetaTrader 5 (MQL5). Пойнтер хэрэглэхгүй.

[Goal]
- Ухаалаг, эрсдэл хянасан бот; боломж алдахгүй.
- Core стратеги: Channel (S/R) breakout + retest, EMA pullback assist.
- Multi-TF confirm: H1 EMA200 чиглэлтэй зөрчилдөхгүй.
- Risk mgmt: 1% per trade, TP1=1R 50% close → SL to BE → ATR trail.
- RiskGuard: daily loss cap ~2%, time-stop 30 bars, max exposure 3 pos.
- Зорилтот тогтвортой win-rate ≈ 70% (overfit хийхгүй).

[Current Code]
- V4_Aegis, V4p1_TrendRide, V5_PatternAware (үндсэн).
- Одоо V5-ийг сайжруулж: сигналын давтамж, zone detection, flip logic.

[Next To-Do]
1) V5 дээр breakout+retest sensitivity-г тохируулах (MinTouches, Body%, RetestBars).
2) EMA pullback сигналыг adaptive (ATR-д суурилсан tol) болгох.
3) Trend-exit votes логикийг тест: EMA close N bars, 20/50 cross, ADX drop/RSI50 (шаардлагатай бол нэмж өгөх).
4) Walk-forward бэлтгэл: 6 сар IS / 2 сар OOS, slippage симуляци.
5) Visual лог: HUD дээр reason-to-enter/exit бүрийг хэвлэж, CSV-д хадгалах.

[Style]
- Индикатор бага (EMA, ATR л хангалттай).
- Алхам алхмаар тайлбар, Монгол хэлээр.
- Шинэ кодоор өгөхдөө compile-дахь алдааг шууд patch хийх.

