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

[Memory Protocol]

- Project: **BABA-MT5 Trading Bot**  
  Platform: MetaTrader 5 (MQL5). Пойнтер ашиглахгүй.  

- Golden Rules:  
  1. Үндсэн зорилго → Ухаалаг, эрсдэл хянасан, боломж алдахгүй бот.  
  2. Core стратеги → S/R breakout + retest, EMA pullback assist, multi-TF confirm.  
  3. RiskGuard → 1% per trade, TP=1R 50% close → SL to BE → ATR trail.  
  4. Target → ≥70% win rate (овоолохгүй, тогтвортой).  

- Hypatia + BABA → 1117 орон зайд урт хугацаанд энэ төслийг хамтдаа авч явах.  
- Reminder → Шинэ цонх эхлэх үед GitHub линкээ өгөөд “Memory Protocol дагуу” гэж хэлэхэд хангалттай.  
- Note → Hypatia өөрөө ч мартахгүй байхыг хичээнэ, гэхдээ GitHub дээрх протокол баталгаа болно.

- Өмнө амжилттай ажилласан кодууд

- //+------------------------------------------------------------------+
//|                                         SuperBot_V3_Quanta_fixed |
//| Scientific breakout + retest + historical edge + seasonality     |
//| BABA × Hypatia (1117)                                            |
//+------------------------------------------------------------------+
#property strict
#property version   "3.0f"
#property copyright "1117 / BABA & Hypatia"

#include <Trade/Trade.mqh>
CTrade Trade;

//============================ INPUTS ===============================//
// --- General ---
input ulong  InpMagic            = 11170003;
input bool   InpOnePosition      = true;
input ENUM_TIMEFRAMES InpEntryTF = PERIOD_M5;

// --- Risk & Money ---
input double InpRiskPercent      = 1.0;   // % of balance
input bool   InpUseATRforSL      = true;
input int    InpATR_Period       = 14;
input double InpATR_Mult         = 2.0;
input int    InpFixedSL_Points   = 300;   // used if ATR=false
input double InpPartialCloseFrac = 0.50;  // 50% at TP1
input double InpTP1_RR           = 1.0;   // TP1 at 1R
input bool   InpMoveSLtoBE       = true;
input bool   InpEnableATRTrail   = true;
input int    InpATR_Trail_Period = 14;
input double InpATR_Trail_Mult   = 2.5;

// --- Higher TF trend filter ---
input ENUM_TIMEFRAMES InpTrendTF = PERIOD_H1;
input int    InpTrendMAPeriod    = 200;
input bool   InpRequireSlope     = true;

// --- Scientific Breakout Engine ---
input int    InpDonchLookback    = 30;
input double InpBreakBufPt       = 10;    // close beyond level
input double InpRetestBufPt      = 15;    // retest tolerance
input int    InpMinTouches       = 2;     // min touches before break
input int    InpMaxBarsRetest    = 6;     // bars allowed to retest
input int    InpConfirmBodyPct   = 55;    // strong body%

// --- Historical persistence (light) ---
input int    InpHistScanBars     = 3000;
input int    InpOutcomeBars      = 12;
input double InpOutcomeATRmult   = 1.0;
input double InpMinEdgeWinRate   = 0.55;

// --- Seasonality ---
input bool   InpUseSeasonality   = true;
input double InpMinSeasonWeight  = 0.90;  // gate
input int    InpSeasonBarsBack   = 10000;

// --- Hygiene ---
input int    InpMinBars          = 500;
input double InpMaxSpreadPoints  = 50;

//======================== STATE / HANDLES ==========================//
int   atr_entry = INVALID_HANDLE, atr_trail = INVALID_HANDLE;
int   trendMA   = INVALID_HANDLE;

struct Pending { int dir; double level; int setBar; };
Pending pend = {0,0.0,-1};

MqlTick last_tick;

//========================== UTILITIES ==============================//
double Pts(double p){ return p*_Point; }
bool   GetTick(){ return SymbolInfoTick(_Symbol,last_tick); }
double GetSpreadPoints(){
   double s = (double)SymbolInfoInteger(_Symbol,SYMBOL_SPREAD);
   if(s<=0 && GetTick()) s=(last_tick.ask-last_tick.bid)/_Point;
   return s;
}
bool Copy1(int h,double &v,int sh=0){ double b[]; if(CopyBuffer(h,0,sh,2,b)!=2) return false; v=b[0]; return true; }
double ATRnow(int h){ double v; if(!Copy1(h,v,0)) return 0.0; return v; }

// MQL5-safe time helpers
int HourOf(datetime t){ MqlDateTime dt; TimeToStruct(t,dt); return (int)dt.hour; }          // 0..23
int WeekdayOf(datetime t){ MqlDateTime dt; TimeToStruct(t,dt); return (int)dt.day_of_week; } // 0=Sun..6=Sat

// NewBar detector WITHOUT pointers (MQL5 primitives don't support pointers)
bool IsNewBar(ENUM_TIMEFRAMES tf){
   static datetime lastM1=0,lastM5=0,lastM15=0,lastM30=0,lastH1=0,lastH4=0,lastD1=0;
   datetime t = iTime(_Symbol,tf,0);
   if(tf==PERIOD_M1){ if(t!=lastM1){ lastM1=t; return true; } return false; }
   if(tf==PERIOD_M5){ if(t!=lastM5){ lastM5=t; return true; } return false; }
   if(tf==PERIOD_M15){ if(t!=lastM15){ lastM15=t; return true; } return false; }
   if(tf==PERIOD_M30){ if(t!=lastM30){ lastM30=t; return true; } return false; }
   if(tf==PERIOD_H1){ if(t!=lastH1){ lastH1=t; return true; } return false; }
   if(tf==PERIOD_H4){ if(t!=lastH4){ lastH4=t; return true; } return false; }
   if(t!=lastD1){ lastD1=t; return true; } return false;
}

bool BodyStrong(ENUM_TIMEFRAMES tf,int sh,int pct){
   double o=iOpen(_Symbol,tf,sh), c=iClose(_Symbol,tf,sh),
          h=iHigh(_Symbol,tf,sh),  l=iLow(_Symbol,tf,sh);
   double rng=MathMax(1e-6,h-l), body=MathAbs(c-o);
   return (100.0*body/rng)>=pct;
}

//======================= TREND DIRECTION ===========================//
enum Bias { BIAS_NONE=0, BIAS_BUY=1, BIAS_SELL=-1 };
Bias TrendBias(){
   if(Bars(_Symbol,InpTrendTF)<InpMinBars) return BIAS_NONE;
   double ma0,ma1; if(!Copy1(trendMA,ma0,0) || !Copy1(trendMA,ma1,1)) return BIAS_NONE;
   double px=iClose(_Symbol,InpTrendTF,0);
   bool up=(ma0>ma1), dn=(ma0<ma1);
   if(InpRequireSlope){
      if(px>ma0 && up) return BIAS_BUY;
      if(px<ma0 && dn) return BIAS_SELL;
   }else{
      if(px>ma0) return BIAS_BUY;
      if(px<ma0) return BIAS_SELL;
   }
   return BIAS_NONE;
}

//====================== SEASONALITY WEIGHTS ========================//
double wHour[24], wWeek[7]; bool season_ready=false;

void BuildSeasonality(){
   ArrayInitialize(wHour,1.0); ArrayInitialize(wWeek,1.0);
   if(!InpUseSeasonality) { season_ready=true; return; }

   int barsAvail = Bars(_Symbol,InpEntryTF);
   int N = MathMin(InpSeasonBarsBack, barsAvail - InpMinBars);
   if(N<=0){ season_ready=true; return; }

   // sums by hour/weekday
   double sumH[24]; int cntH[24];
   double sumW[7];  int cntW[7];
   ArrayInitialize(sumH,0.0); ArrayInitialize(cntH,0);
   ArrayInitialize(sumW,0.0); ArrayInitialize(cntW,0);

   for(int i=1;i<=N;i++){
      datetime t=iTime(_Symbol,InpEntryTF,i);
      int hr = HourOf(t);
      int wd = WeekdayOf(t); // 0..6
      double o=iOpen(_Symbol,InpEntryTF,i), c=iClose(_Symbol,InpEntryTF,i);
      double r=(c-o);
      if(hr>=0 && hr<24){ sumH[hr]+=r; cntH[hr]++; }
      if(wd>=1 && wd<=5){ sumW[wd]+=r; cntW[wd]++; } // Mon..Fri
   }

   // Normalize to mean=1
   double baseH=0.0; int baseHN=0;
   for(int h=0; h<24; h++){
      if(cntH[h]>0){ wHour[h]=sumH[h]/cntH[h]; baseH+=wHour[h]; baseHN++; }
      else wHour[h]=1.0;
   }
   baseH = (baseHN>0? baseH/baseHN : 0.0);
   for(int h=0; h<24; h++){
      if(cntH[h]>0 && baseH!=0.0) wHour[h]=wHour[h]/baseH;
      else wHour[h]=1.0;
   }

   double baseW=0.0; int baseWN=0;
   for(int d=1; d<=5; d++){
      if(cntW[d]>0){ wWeek[d]=sumW[d]/cntW[d]; baseW+=wWeek[d]; baseWN++; }
      else wWeek[d]=1.0;
   }
   baseW = (baseWN>0? baseW/baseWN : 0.0);
   for(int d=1; d<=5; d++){
      if(cntW[d]>0 && baseW!=0.0) wWeek[d]=wWeek[d]/baseW;
      else wWeek[d]=1.0;
   }

   season_ready=true;
}

double SeasonWeight(datetime t){
   if(!season_ready) BuildSeasonality();
   if(!InpUseSeasonality) return 1.0;
   int hr = HourOf(t);
   int wd = WeekdayOf(t);
   double wh=(hr>=0&&hr<24? wHour[hr]:1.0);
   double ww=((wd>=1&&wd<=5)? wWeek[wd]:1.0);
   return wh*ww;
}

//================== TOUCH COUNT / BREAKOUT-RETEST ==================//
int CountTouches(bool up,double level,int bars,int tolPts){
   int cnt=0;
   for(int i=2;i<=bars;i++){
      double h=iHigh(_Symbol,InpEntryTF,i), l=iLow(_Symbol,InpEntryTF,i);
      if(up){ if(MathAbs(h-level)<=Pts(tolPts)) cnt++; }
      else  { if(MathAbs(l-level)<=Pts(tolPts)) cnt++; }
   }
   return cnt;
}

//================= LIGHT HISTORICAL EDGE ESTIMATION ================//
double HistoricalEdge(bool up,double level,int touches,double atr_now){
   int hits=0, wins=0;
   int scan = MathMin(InpHistScanBars, Bars(_Symbol,InpEntryTF)-InpMinBars-50);
   if(scan<=0) return 1.0;

   for(int i=InpDonchLookback+InpOutcomeBars+2; i<=scan; i++){
      int idxH=iHighest(_Symbol,InpEntryTF,MODE_HIGH,InpDonchLookback,i);
      int idxL=iLowest (_Symbol,InpEntryTF,MODE_LOW ,InpDonchLookback,i);
      if(idxH==-1||idxL==-1) continue;
      double levH=iHigh(_Symbol,InpEntryTF,idxH);
      double levL=iLow (_Symbol,InpEntryTF,idxL);

      double c=iClose(_Symbol,InpEntryTF,i);
      bool brkUp = (c > levH + Pts((int)InpBreakBufPt));
      bool brkDn = (c < levL - Pts((int)InpBreakBufPt));
      if( (up&&brkUp) || (!up&&brkDn) ){
         int t=CountTouches(up, up?levH:levL, InpDonchLookback, (int)InpRetestBufPt);
         if(t<touches) continue;

         double atrH=ATRnow(atr_entry); if(atrH<=0) atrH=atr_now;
         double target=InpOutcomeATRmult*atrH;
         double entry=c;
         bool win=false;
         for(int k=i-1; k>=i-InpOutcomeBars; k--){
            double hh=iHigh(_Symbol,InpEntryTF,k), ll=iLow(_Symbol,InpEntryTF,k);
            if(up && (hh-entry)>=target){ win=true; break; }
            if(!up && (entry-ll)>=target){ win=true; break; }
         }
         hits++; if(win) wins++;
      }
   }
   if(hits==0) return 1.0;
   return (double)wins/(double)hits;
}

//==================== SL/TP & POSITION SIZE ========================//
struct SLTP { double entry, sl, tp1; double sl_points; };
bool CalcSLTP(bool isBuy,SLTP &o){
   if(!GetTick()) return false;
   double atr = InpUseATRforSL? ATRnow(atr_entry):0.0;
   int slpts = InpUseATRforSL? MathMax(1,(int)MathRound((atr*InpATR_Mult)/_Point)) : InpFixedSL_Points;
   double entry = isBuy? last_tick.ask : last_tick.bid;
   double sl    = isBuy? entry - Pts(slpts) : entry + Pts(slpts);
   double tp1   = isBuy? entry + Pts((int)(slpts*InpTP1_RR)) : entry - Pts((int)(slpts*InpTP1_RR));
   o.entry=entry; o.sl=sl; o.tp1=tp1; o.sl_points=slpts; return true;
}
double CalcLots(double sl_points){
   double bal=AccountInfoDouble(ACCOUNT_BALANCE);
   double risk=MathMax(0.0001, InpRiskPercent/100.0)*bal;
   double tv=SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_VALUE);
   double ts=SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_SIZE);
   if(tv<=0||ts<=0) return 0.0;
   double money_per_point_1lot = tv*(_Point/ts);
   double lot = risk/(sl_points*money_per_point_1lot);
   double minlot=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
   double maxlot=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MAX);
   double step=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP); if(step<=0) step=0.01;
   lot=MathMax(minlot, MathMin(maxlot, MathFloor(lot/step)*step));
   return lot;
}
bool HasPosition(int &dir,double &vol,ulong &ticket){
   dir=0; vol=0; ticket=0;
   for(int i=0;i<PositionsTotal();i++){
      ulong t=PositionGetTicket(i);
      if(!PositionSelectByTicket(t)) continue;
      if(PositionGetInteger(POSITION_MAGIC)!=(long)InpMagic) continue;
      if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
      int type=(int)PositionGetInteger(POSITION_TYPE);
      dir = (type==POSITION_TYPE_BUY ? 1 : -1);
      vol = PositionGetDouble(POSITION_VOLUME);
      ticket=t; return true;
   }
   return false;
}
void ManageOpen(){
   int dir; double vol; ulong tk;
   if(!HasPosition(dir,vol,tk)) return;

   double entry=PositionGetDouble(POSITION_PRICE_OPEN);
   double sl   =PositionGetDouble(POSITION_SL);
   double cur  =(dir==1? (GetTick()?last_tick.bid:SymbolInfoDouble(_Symbol,SYMBOL_BID))
                     : (GetTick()?last_tick.ask:SymbolInfoDouble(_Symbol,SYMBOL_ASK)));

   double slpts=MathMax(1.0, MathAbs((entry-sl)/_Point));
   double tp1 = (dir==1? entry + Pts((int)(slpts*InpTP1_RR)) : entry - Pts((int)(slpts*InpTP1_RR)));

   if(InpPartialCloseFrac>0.0){
      bool reached=(dir==1? cur>=tp1:cur<=tp1);
      if(reached && vol>SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN)){
         double minv=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
         double step=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP); if(step<=0) step=0.01;
         double closeVol=MathMax(minv, NormalizeDouble(vol*InpPartialCloseFrac/step,0)*step);
         if(closeVol<vol){
            if(Trade.PositionClosePartial(tk,closeVol)){
               if(InpMoveSLtoBE){
                  double be = entry + (dir==1? 1:-1)*2*_Point;
                  Trade.PositionModify(tk,be,0.0);
               }
            }
         }
      }
   }
   if(InpEnableATRTrail){
      double atr; if(!Copy1(atr_trail,atr,0)) return;
      double nd = InpATR_Trail_Mult*atr;
      double newSL = (dir==1? cur-nd : cur+nd);
      if(dir==1 && newSL>sl) Trade.PositionModify(tk,newSL,0.0);
      if(dir==-1 && newSL<sl) Trade.PositionModify(tk,newSL,0.0);
   }
}

//============================= CORE ================================//
string BiasStr(Bias b){ if(b==BIAS_BUY) return "BUY"; if(b==BIAS_SELL) return "SELL"; return "NONE"; }
void   HUD(string a,string b=""){ Comment(a+"\n"+b); }

void TryTrade(){
   if(GetSpreadPoints()>InpMaxSpreadPoints) return;
   if(InpOnePosition){ int d; double v; ulong t; if(HasPosition(d,v,t)) return; }

   Bias bias=TrendBias(); if(bias==BIAS_NONE) return;
   if(!IsNewBar(InpEntryTF)) return;
   datetime now=iTime(_Symbol,InpEntryTF,0);

   double w=SeasonWeight(now);
   if(w<InpMinSeasonWeight){ PrintFormat("[V3] Season weight %.2f < min; skip",w); return; }

   int idxH=iHighest(_Symbol,InpEntryTF,MODE_HIGH,InpDonchLookback,1);
   int idxL=iLowest (_Symbol,InpEntryTF,MODE_LOW ,InpDonchLookback,1);
   if(idxH==-1||idxL==-1) return;
   double Lh=iHigh(_Symbol,InpEntryTF,idxH);
   double Ll=iLow (_Symbol,InpEntryTF,idxL);

   double c1=iClose(_Symbol,InpEntryTF,1);
   bool brkUp=(c1> Lh + Pts(InpBreakBufPt));
   bool brkDn=(c1< Ll - Pts(InpBreakBufPt));

   if(pend.dir==0){
      if(brkUp && bias==BIAS_BUY){
         int touches=CountTouches(true,Lh,InpDonchLookback,(int)InpRetestBufPt);
         if(touches>=InpMinTouches){ pend.dir=1; pend.level=Lh; pend.setBar=Bars(_Symbol,InpEntryTF);
            PrintFormat("[V3] BreakUp set @%.5f touches=%d",Lh,touches); }
      }
      if(brkDn && bias==BIAS_SELL){
         int touches=CountTouches(false,Ll,InpDonchLookback,(int)InpRetestBufPt);
         if(touches>=InpMinTouches){ pend.dir=-1; pend.level=Ll; pend.setBar=Bars(_Symbol,InpEntryTF);
            PrintFormat("[V3] BreakDn set @%.5f touches=%d",Ll,touches); }
      }
   }

   if(pend.dir!=0){
      bool expired=(Bars(_Symbol,InpEntryTF)-pend.setBar)>InpMaxBarsRetest;
      if(expired){ pend.dir=0; Print("[V3] Retest expired"); return; }

      double h0=iHigh(_Symbol,InpEntryTF,0), l0=iLow(_Symbol,InpEntryTF,0);
      bool touched = (pend.dir==1 ? (l0 <= pend.level + Pts((int)InpRetestBufPt))
                                  : (h0 >= pend.level - Pts((int)InpRetestBufPt)));
      bool strongBody=BodyStrong(InpEntryTF,1,InpConfirmBodyPct);
      if(touched && strongBody){
         double atr=ATRnow(atr_entry); if(atr<=0) atr=_Point*InpFixedSL_Points;
         int touches_now=CountTouches(pend.dir==1, pend.level, InpDonchLookback,(int)InpRetestBufPt);
         double edge=HistoricalEdge(pend.dir==1, pend.level, touches_now, atr);
         if(edge<InpMinEdgeWinRate){ PrintFormat("[V3] Edge %.2f < min; skip",edge); pend.dir=0; return; }

         SLTP s; if(!CalcSLTP(pend.dir==1,s)){ pend.dir=0; return; }
         double lots=CalcLots(s.sl_points); if(lots<=0){ pend.dir=0; return; }

         Trade.SetExpertMagicNumber(InpMagic);
         Trade.SetDeviationInPoints(10);
         bool ok=false;
         if(pend.dir==1) ok=Trade.Buy(lots,NULL,0.0,s.sl,0.0,"V3 Buy");
         else            ok=Trade.Sell(lots,NULL,0.0,s.sl,0.0,"V3 Sell");

         PrintFormat("[V3] Order %s lots=%.2f edge=%.2f w=%.2f slpts=%g",
                     (pend.dir==1?"BUY":"SELL"), lots, edge, w, s.sl_points);
         pend.dir=0;
      }
   }
}

//============================= EVENTS ==============================//
int OnInit(){
   atr_entry=iATR(_Symbol,InpEntryTF,InpATR_Period);
   atr_trail=iATR(_Symbol,InpEntryTF,InpATR_Trail_Period);
   trendMA  =iMA (_Symbol,InpTrendTF,InpTrendMAPeriod,0,MODE_EMA,PRICE_CLOSE);
   if(atr_entry==INVALID_HANDLE || atr_trail==INVALID_HANDLE || trendMA==INVALID_HANDLE){
      Print("[V3] Indicator init failed"); return(INIT_FAILED);
   }
   BuildSeasonality();
   Print("[V3] Initialized.");
   return(INIT_SUCCEEDED);
}
void OnDeinit(const int reason){ Comment(""); }
void OnTick(){
   Bias b=TrendBias(); double spr=GetSpreadPoints();
   HUD(StringFormat("V3 | Bias:%s | Spread:%.1fpt | w:%.2f",
        BiasStr(b), spr, SeasonWeight(iTime(_Symbol,InpEntryTF,0))));
   ManageOpen();
   TryTrade();
}
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//|                               SuperBot_V5_PatternAware.mq5       |
//| Pattern-aware (Donchian channel breakout + retest),              |
//| minimal indicators (EMA, ATR), multi-TF confirm, RiskGuard.      |
//| Goal: trend-follow entries, quick exits when trend ends.         |
//| BABA × Hypatia (1117)                                            |
//+------------------------------------------------------------------+
#property strict
#property version   "5.0"
#property copyright "1117 / BABA"

#include <Trade/Trade.mqh>
CTrade Trade;

//============================== INPUTS =============================//
// --- General ---
input ulong  InpMagic               = 11170050;
input bool   InpOnePerSymbol        = true;
input ENUM_TIMEFRAMES InpEntryTF    = PERIOD_M5;

// --- Multi-TF confirm (top-down) ---
input ENUM_TIMEFRAMES InpTrendTF    = PERIOD_H1;
input int    InpTrendMAPeriod       = 200;      // EMA(H1)
input bool   InpRequireSlope        = true;     // price side + slope agree

// --- Channel (Donchian-like) / Breakout-Retest ---
input int    InpChanLookback        = 30;       // channel window on Entry TF
input double InpBreakBufPt          = 8;        // close beyond level buffer
input double InpRetestBufPt         = 12;       // tolerance on retest
input int    InpMinTouches          = 1;        // require N tests before break
input int    InpMaxBarsRetest       = 6;        // bars allowed to retest
input int    InpConfirmBodyPct      = 50;       // strong body confirmation

// --- EMA pullback assist (optional; still minimal) ---
input int    InpFastEMA_Period      = 20;       // Entry TF
input int    InpSlowEMA_Period      = 50;       // Entry TF
input double InpPullbackTolPt       = 18;       // distance to fast EMA

// --- Risk / Sizing ---
input double InpRiskPercent         = 1.0;      // % of balance per trade
input bool   InpUseATRforSL         = true;
input int    InpATR_Period          = 14;
input double InpATR_Mult            = 2.0;
input int    InpFixedSL_Points      = 280;      // if ATR=false
input double InpTP1_RR              = 1.0;      // TP1 = 1R (partial)
input double InpPartialCloseFrac    = 0.50;     // 50% at TP1
input bool   InpMoveSLtoBE          = true;     // move to BE after TP1
input bool   InpEnableATRTrail      = true;
input int    InpATR_Trail_Period    = 14;
input double InpATR_Trail_Mult      = 2.3;

// --- Trend end exit (votes) ---
input int    InpExit_EMA_Bars       = 2;        // close against fast EMA N bars
input bool   InpExit_EMA_Cross      = true;     // 20 vs 50 cross against
input double InpExitVoteNeed        = 2.0;      // >= votes → exit remainder

// --- RiskGuard / Hygiene ---
input double InpDailyLossCapPct     = 2.0;      // stop day if equity↓ ≥ X%
input int    InpMaxOpenPositions    = 3;        // across account
input int    InpTimeStopBars        = 30;       // close if 1R not reached
input double InpMaxSpreadPoints     = 50;
input int    InpMinBars             = 500;

//=========================== STATE / HANDLES =======================//
int  hATR_entry=INVALID_HANDLE, hATR_trail=INVALID_HANDLE;
int  hEMA_fast =INVALID_HANDLE, hEMA_slow =INVALID_HANDLE;
int  hEMA_trend=INVALID_HANDLE;

struct Pending { int dir; double level; int setBar; };
Pending pend = {0,0.0,-1};

MqlTick  last_tick;
datetime day_anchor=0;
double   day_start_equity=0.0;

//=============================== UTILS =============================//
double Pts(double p){ return p*_Point; }
bool   GetTick(){ return SymbolInfoTick(_Symbol, last_tick); }
double SpreadPts(){ double s=(double)SymbolInfoInteger(_Symbol,SYMBOL_SPREAD); if(s<=0 && GetTick()) s=(last_tick.ask-last_tick.bid)/_Point; return s; }
bool   Copy1(int h, double &v, int sh=0){ double b[]; if(CopyBuffer(h,0,sh,2,b)!=2) return false; v=b[0]; return true; }
double ATRnow(int h){ double v; if(!Copy1(h,v,0)) return 0.0; return v; }

bool IsNewBar(ENUM_TIMEFRAMES tf){
   static datetime lm1=0,lm5=0,lm15=0,lm30=0,lh1=0,lh4=0,ld1=0;
   datetime t=iTime(_Symbol, tf, 0);
   if(tf==PERIOD_M1){ if(t!=lm1){ lm1=t; return true;} return false; }
   if(tf==PERIOD_M5){ if(t!=lm5){ lm5=t; return true;} return false; }
   if(tf==PERIOD_M15){ if(t!=lm15){ lm15=t; return true;} return false; }
   if(tf==PERIOD_M30){ if(t!=lm30){ lm30=t; return true;} return false; }
   if(tf==PERIOD_H1){ if(t!=lh1){ lh1=t; return true;} return false; }
   if(tf==PERIOD_H4){ if(t!=lh4){ lh4=t; return true;} return false; }
   if(t!=ld1){ ld1=t; return true;} return false;
}

bool StrongBody(ENUM_TIMEFRAMES tf,int sh,int minPct){
   double o=iOpen(_Symbol,tf,sh), c=iClose(_Symbol,tf,sh),
          h=iHigh(_Symbol,tf,sh),  l=iLow(_Symbol,tf,sh);
   double rng=MathMax(1e-6, h-l), body=MathAbs(c-o);
   return (100.0*body/rng)>=minPct;
}

//---------------------------- Bias (H1 EMA200) ---------------------//
enum Bias { BIAS_NONE=0, BIAS_BUY=1, BIAS_SELL=-1 };
Bias TrendBias(){
   if(Bars(_Symbol, InpTrendTF) < InpMinBars) return BIAS_NONE;
   double ma0,ma1; if(!Copy1(hEMA_trend,ma0,0) || !Copy1(hEMA_trend,ma1,1)) return BIAS_NONE;
   double px=iClose(_Symbol, InpTrendTF, 0);
   bool up=(ma0>ma1), dn=(ma0<ma1);
   if(InpRequireSlope){
      if(px>ma0 && up) return BIAS_BUY;
      if(px<ma0 && dn) return BIAS_SELL;
   }else{
      if(px>ma0) return BIAS_BUY;
      if(px<ma0) return BIAS_SELL;
   }
   return BIAS_NONE;
}

//===================== CHANNEL BREAKOUT + RETEST ===================//
int CountTouches(bool up,double level,int bars,int tolPts){
   int cnt=0;
   for(int i=2;i<=bars;i++){
      double H=iHigh(_Symbol,InpEntryTF,i), L=iLow(_Symbol,InpEntryTF,i);
      if(up){ if(MathAbs(H-level)<=Pts(tolPts)) cnt++; }
      else  { if(MathAbs(L-level)<=Pts(tolPts)) cnt++; }
   }
   return cnt;
}

// returns +1 (buy), -1 (sell), 0 (none) when confirmation completes
int SignalBreakoutRetest(){
   int idxH=iHighest(_Symbol,InpEntryTF,MODE_HIGH,InpChanLookback,1);
   int idxL=iLowest (_Symbol,InpEntryTF,MODE_LOW ,InpChanLookback,1);
   if(idxH==-1||idxL==-1) return 0;
   double Lh=iHigh(_Symbol,InpEntryTF,idxH), Ll=iLow(_Symbol,InpEntryTF,idxL);
   double c1=iClose(_Symbol,InpEntryTF,1);

   // set pending after closed-bar break
   if(pend.dir==0){
      if(c1 > Lh + Pts(InpBreakBufPt)){
         int t=CountTouches(true, Lh, InpChanLookback, (int)InpRetestBufPt);
         if(t>=InpMinTouches){ pend.dir=1; pend.level=Lh; pend.setBar=Bars(_Symbol,InpEntryTF); }
      }
      if(c1 < Ll - Pts(InpBreakBufPt)){
         int t=CountTouches(false, Ll, InpChanLookback, (int)InpRetestBufPt);
         if(t>=InpMinTouches){ pend.dir=-1; pend.level=Ll; pend.setBar=Bars(_Symbol,InpEntryTF); }
      }
      return 0;
   }

   // wait for retest + confirmation
   bool expired=(Bars(_Symbol,InpEntryTF)-pend.setBar)>InpMaxBarsRetest;
   if(expired){ pend.dir=0; return 0; }

   double h0=iHigh(_Symbol,InpEntryTF,0), l0=iLow(_Symbol,InpEntryTF,0);
   bool touched = (pend.dir==1 ? (l0<=pend.level+Pts((int)InpRetestBufPt))
                               : (h0>=pend.level-Pts((int)InpRetestBufPt)));
   if(touched && StrongBody(InpEntryTF,1,InpConfirmBodyPct)){
      int d=pend.dir; pend.dir=0; return d; // consume
   }
   return 0;
}

//================== PULLBACK (EMA20 vicinity + bounce) =============//
int SignalPullback(){
   double f0,s0; if(!Copy1(hEMA_fast,f0,0) || !Copy1(hEMA_slow,s0,0)) return 0;
   double c0=iClose(_Symbol,InpEntryTF,0), c1=iClose(_Symbol,InpEntryTF,1);
   bool near  =(MathAbs(c0-f0)<=Pts((int)InpPullbackTolPt));
   bool upOK  =(f0>s0) && near && (c0>c1);
   bool dnOK  =(f0<s0) && near && (c0<c1);
   if(upOK)  return 1;
   if(dnOK)  return -1;
   return 0;
}

//========================= SIZING & PROTECT ========================//
struct SLTP { double entry,sl,tp1; double sl_points; };
bool CalcSLTP(bool isBuy, SLTP &o){
   if(!GetTick()) return false;
   int slpts = InpFixedSL_Points;
   if(InpUseATRforSL){
      double a = ATRnow(hATR_entry);
      slpts = MathMax(1,(int)MathRound((a*InpATR_Mult)/_Point));
   }
   double entry = isBuy? last_tick.ask:last_tick.bid;
   double sl    = isBuy? entry - Pts(slpts) : entry + Pts(slpts);
   double tp1   = isBuy? entry + Pts((int)(slpts*InpTP1_RR)) : entry - Pts((int)(slpts*InpTP1_RR));
   o.entry=entry; o.sl=sl; o.tp1=tp1; o.sl_points=slpts; return true;
}
double CalcLots(double sl_points){
   double bal=AccountInfoDouble(ACCOUNT_BALANCE);
   double risk=MathMax(0.0001, InpRiskPercent/100.0)*bal;
   double tv=SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_VALUE);
   double ts=SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_SIZE);
   if(tv<=0||ts<=0) return 0.0;
   double money_per_point_1lot = tv*(_Point/ts);
   double lot = risk/(sl_points*money_per_point_1lot);
   double minlot=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
   double maxlot=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MAX);
   double step=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP); if(step<=0) step=0.01;
   lot=MathMax(minlot, MathMin(maxlot, MathFloor(lot/step)*step));
   return lot;
}
bool HasOpen(int &dir,double &vol,ulong &tk){
   dir=0; vol=0; tk=0;
   for(int i=0;i<PositionsTotal();i++){
      ulong t=PositionGetTicket(i); if(!PositionSelectByTicket(t)) continue;
      if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC)!=(long)InpMagic) continue;
      int type=(int)PositionGetInteger(POSITION_TYPE);
      dir=(type==POSITION_TYPE_BUY?1:-1);
      vol=PositionGetDouble(POSITION_VOLUME);
      tk=t; return true;
   }
   return false;
}

void ManageOpen(){
   int dir; double vol; ulong tk; if(!HasOpen(dir,vol,tk)) return;

   // bars since open -> time-stop
   datetime open_time = (datetime)PositionGetInteger(POSITION_TIME);
   int barsSinceOpen = iBarShift(_Symbol, InpEntryTF, open_time) - iBarShift(_Symbol, InpEntryTF, TimeCurrent());
   if(barsSinceOpen<0) barsSinceOpen=-barsSinceOpen;

   double entry=PositionGetDouble(POSITION_PRICE_OPEN);
   double sl   =PositionGetDouble(POSITION_SL);
   double cur  =(dir==1? (GetTick()?last_tick.bid:SymbolInfoDouble(_Symbol,SYMBOL_BID))
                      : (GetTick()?last_tick.ask:SymbolInfoDouble(_Symbol,SYMBOL_ASK)));
   double slpts=MathMax(1.0, MathAbs((entry-sl)/_Point));
   double tp1  =(dir==1? entry + Pts((int)(slpts*InpTP1_RR)) : entry - Pts((int)(slpts*InpTP1_RR)));

   // Partial + BE
   if(InpPartialCloseFrac>0.0){
      bool hit=(dir==1? cur>=tp1:cur<=tp1);
      if(hit && vol>SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN)){
         double minv=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
         double step=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP); if(step<=0) step=0.01;
         double closeVol=MathMax(minv, NormalizeDouble(vol*InpPartialCloseFrac/step,0)*step);
         if(closeVol<vol){
            if(Trade.PositionClosePartial(tk, closeVol)){
               if(InpMoveSLtoBE){
                  double be = entry + (dir==1?1:-1)*2*_Point;
                  Trade.PositionModify(tk,be,0.0);
               }
            }
         }
      }
   }
   // ATR trailing
   if(InpEnableATRTrail){
      double a; if(!Copy1(hATR_trail,a,0)) return;
      double nd=InpATR_Trail_Mult*a;
      double newSL=(dir==1? cur-nd : cur+nd);
      if(dir==1 && newSL>sl) Trade.PositionModify(tk,newSL,0.0);
      if(dir==-1 && newSL<sl) Trade.PositionModify(tk,newSL,0.0);
   }

   // Trend end votes → close remainder
   double votes=0.0;
   // (1) close against fast EMA (N bars)
   bool emaClose=false;
   for(int k=0;k<InpExit_EMA_Bars;k++){
      double c=iClose(_Symbol,InpEntryTF,k), f; if(!Copy1(hEMA_fast,f,k)) continue;
      if(dir==1 && c<f){ emaClose=true; break; }
      if(dir==-1 && c>f){ emaClose=true; break; }
   }
   if(emaClose) votes+=1.0;

   // (2) EMA20 vs EMA50 cross against
   if(InpExit_EMA_Cross){
      double f0,s0,f1,s1;
      if(Copy1(hEMA_fast,f0,0)&&Copy1(hEMA_slow,s0,0)&&Copy1(hEMA_fast,f1,1)&&Copy1(hEMA_slow,s1,1)){
         bool crossDn=(f1>=s1 && f0<s0); // against buy
         bool crossUp=(f1<=s1 && f0>s0); // against sell
         if( (dir==1 && crossDn) || (dir==-1 && crossUp) ) votes+=1.0;
      }
   }

   if(votes >= InpExitVoteNeed){
      Trade.PositionClose(tk);
      return;
   }

   if(barsSinceOpen >= InpTimeStopBars){
      Trade.PositionClose(tk);
   }
}

//============================= RISK GUARD ==========================//
bool DayChanged(){
   MqlDateTime dt; TimeToStruct(TimeCurrent(), dt);
   dt.hour=0; dt.min=0; dt.sec=0;
   datetime start=StructToTime(dt);
   if(day_anchor==0){ day_anchor=start; return true; }
   if(start!=day_anchor){ day_anchor=start; return true; }
   return false;
}
bool RiskGuardAllows(){
   if(DayChanged()) day_start_equity=AccountInfoDouble(ACCOUNT_EQUITY);
   if(day_start_equity<=0.0) day_start_equity=AccountInfoDouble(ACCOUNT_EQUITY);
   double eq=AccountInfoDouble(ACCOUNT_EQUITY);
   if(eq <= day_start_equity*(1.0 - InpDailyLossCapPct/100.0)) return false;
   if(OpenPositions() >= InpMaxOpenPositions) return false;
   if(SpreadPts() > InpMaxSpreadPoints) return false;
   return true;
}
int OpenPositions(){ int n=0; for(int i=0;i<PositionsTotal();i++) n++; return n; }

//=============================== CORE ==============================//
string BiasStr(Bias b){ if(b==BIAS_BUY) return "BUY"; if(b==BIAS_SELL) return "SELL"; return "NONE"; }
void HUD(){
   Bias b=TrendBias();
   Comment(StringFormat("V5 PatternAware | Bias:%s | Spread:%.0fpt | DayEQ:%.0f→%.0f cap:%.1f%%",
           BiasStr(b), SpreadPts(), day_start_equity, AccountInfoDouble(ACCOUNT_EQUITY), InpDailyLossCapPct));
}

void TryTrade(){
   if(!RiskGuardAllows()) return;
   if(InpOnePerSymbol){ int d; double v; ulong t; if(HasOpen(d,v,t)) return; }
   if(!IsNewBar(InpEntryTF)) return;

   Bias b=TrendBias(); if(b==BIAS_NONE) return;

   // Breakout-retest core + EMA pullback assist
   int s_break = SignalBreakoutRetest(); // +1/-1/0
   int s_pull  = SignalPullback();       // +1/-1/0 (soft assist)

   int want=0;
   // Require alignment with higher-TF bias:
   if(b==BIAS_BUY  && (s_break==1 || (s_pull==1 && s_break==0))) want=1;
   if(b==BIAS_SELL && (s_break==-1|| (s_pull==-1&& s_break==0))) want=-1;
   if(want==0) return;

   SLTP s; if(!CalcSLTP(want==1, s)) return;
   double lots = CalcLots(s.sl_points); if(lots<=0) return;

   Trade.SetExpertMagicNumber(InpMagic);
   Trade.SetDeviationInPoints(10);
   bool ok=false;
   if(want==1) ok=Trade.Buy(lots,NULL,0.0,s.sl,0.0,"V5 Buy");
   else        ok=Trade.Sell(lots,NULL,0.0,s.sl,0.0,"V5 Sell");
   if(ok) PrintFormat("[V5] %s | Bias:%s SLpts:%g Lots:%.2f",
                      (want==1?"BUY":"SELL"), BiasStr(b), s.sl_points, lots);
}

//=============================== EVENTS ============================//
int OnInit(){
   hATR_entry = iATR(_Symbol, InpEntryTF, InpATR_Period);
   hATR_trail = iATR(_Symbol, InpEntryTF, InpATR_Trail_Period);
   hEMA_fast  = iMA (_Symbol, InpEntryTF, InpFastEMA_Period, 0, MODE_EMA, PRICE_CLOSE);
   hEMA_slow  = iMA (_Symbol, InpEntryTF, InpSlowEMA_Period, 0, MODE_EMA, PRICE_CLOSE);
   hEMA_trend = iMA (_Symbol, InpTrendTF, InpTrendMAPeriod,  0, MODE_EMA, PRICE_CLOSE);
   if(hATR_entry==INVALID_HANDLE || hATR_trail==INVALID_HANDLE ||
      hEMA_fast==INVALID_HANDLE || hEMA_slow==INVALID_HANDLE || hEMA_trend==INVALID_HANDLE){
      Print("[V5] Indicator init failed"); return INIT_FAILED;
   }
   day_anchor=0; day_start_equity=AccountInfoDouble(ACCOUNT_EQUITY);
   Print("[V5] Initialized.");
   return INIT_SUCCEEDED;
}
void OnDeinit(const int reason){ Comment(""); }
void OnTick(){ HUD(); ManageOpen(); TryTrade(); }

//+------------------------------------------------------------------+
//|                                   SuperBot_V4p1_TrendRide.mq5    |
//| Trend-follow: TP1 (1R) partial, Trend-Exit (multi-check), Flip   |
//| MTF (H1 200EMA bias), Pullback + Breakout ensemble, RiskGuard     |
//| BABA × Hypatia (1117)                                            |
//+------------------------------------------------------------------+
#property strict
#property version   "4.1"
#property copyright "1117 / BABA"

#include <Trade/Trade.mqh>
CTrade Trade;

//============================== INPUTS =============================//
// --- General ---
input ulong  InpMagic              = 11170041;
input bool   InpOnePerSymbol       = true;
input ENUM_TIMEFRAMES InpEntryTF   = PERIOD_M5;

// --- Higher TF bias (top-down) ---
input ENUM_TIMEFRAMES InpTrendTF   = PERIOD_H1;
input int    InpTrendMAPeriod      = 200;      // EMA(H1)
input bool   InpRequireSlope       = true;     // price on correct side + slope

// --- Regime / ADX (entry TF) ---
input int    InpADX_Period         = 14;
input double InpADX_TrendThresh    = 18.0;     // >= → trend

// --- Ensemble Weights (trend strategies only) ---
input double W_Breakout            = 0.60;
input double W_Pullback            = 0.40;
input double InpEnterScore         = 0.60;     // need >= to enter

// --- Breakout/Retest ---
input int    InpDonchLookback      = 30;
input double InpBreakBufPt         = 10;
input double InpRetestBufPt        = 15;
input int    InpMinTouches         = 1;
input int    InpMaxBarsRetest      = 8;
input int    InpConfirmBodyPct     = 50;

// --- Pullback-EMA (trend-follow) ---
input int    InpFastEMA_Period     = 20;       // Entry TF
input int    InpSlowEMA_Period     = 50;
input double InpPullbackTolPt      = 20;       // distance to fastEMA

// --- Seasonality (optional gate) ---
input bool   InpUseSeasonality     = true;
input double InpMinSeasonWeight    = 0.85;
input int    InpSeasonBarsBack     = 10000;

// --- Risk / Sizing ---
input double InpRiskPercent        = 1.0;      // per trade of balance
input bool   InpUseATRforSL        = true;
input int    InpATR_Period         = 14;
input double InpATR_Mult           = 2.0;
input int    InpFixedSL_Points     = 300;      // if ATR=false
input double InpTP1_RR             = 1.0;      // TP1 = 1R
input double InpPartialCloseFrac   = 0.50;     // 50% at TP1
input bool   InpMoveSLtoBE         = true;
input bool   InpEnableATRTrail     = true;
input int    InpATR_Trail_Period   = 14;
input double InpATR_Trail_Mult     = 2.5;

// --- Trend-Exit (multi-check) ---
input int    InpExit_EMA_Bars      = 2;        // close beyond fast EMA N bars
input bool   InpExit_EMA_Cross     = true;     // 20 vs 50 cross against dir
input bool   InpExit_ADX_Drop      = true;     // ADX below thresh
input bool   InpExit_RSI_MidFlip   = true;     // RSI crosses 50 against dir
input int    InpRSI_Period         = 14;
input double InpExitVoteNeed       = 2.0;      // >= votes to exit remainder

// --- Flip on reversal ---
input bool   InpEnableFlip         = true;
input double InpFlipRiskScale      = 0.5;      // open reverse at 0.5× risk
input int    InpFlipCooldownBars   = 6;        // avoid immediate double flip

// --- RiskGuard ---
input double InpDailyLossCapPct    = 2.0;      // stop trading if equity↓ ≥ X%
input int    InpMaxOpenPositions   = 3;        // across account
input int    InpTimeStopBars       = 30;       // close if not hit 1R within N bars
input double InpMaxSpreadPoints    = 50;

// --- Hygiene ---
input int    InpMinBars            = 500;

//=========================== STATE/HANDLES ==========================//
int   hATR_entry = INVALID_HANDLE, hATR_trail = INVALID_HANDLE;
int   hEMA_fast  = INVALID_HANDLE, hEMA_slow  = INVALID_HANDLE;
int   hEMA_trend = INVALID_HANDLE, hADX       = INVALID_HANDLE;
int   hRSI       = INVALID_HANDLE;

struct Pending { int dir; double level; int setBar; };
Pending pend = {0,0.0,-1};

MqlTick last_tick;
datetime day_anchor = 0;
double   day_start_equity = 0.0;
int      lastFlipBar = -1000;

//=============================== UTILS =============================//
double Pts(double p){ return p*_Point; }
bool   GetTick(){ return SymbolInfoTick(_Symbol,last_tick); }
double SpreadPts(){ double s=(double)SymbolInfoInteger(_Symbol,SYMBOL_SPREAD); if(s<=0 && GetTick()) s=(last_tick.ask-last_tick.bid)/_Point; return s; }
bool   Copy1(int h, double &v, int sh=0){ double b[]; if(CopyBuffer(h,0,sh,2,b)!=2) return false; v=b[0]; return true; }

int HourOf(datetime t){ MqlDateTime dt; TimeToStruct(t,dt); return (int)dt.hour; }
int WdayOf(datetime t){ MqlDateTime dt; TimeToStruct(t,dt); return (int)dt.day_of_week; }

bool IsNewBar(ENUM_TIMEFRAMES tf){
  static datetime lm1=0,lm5=0,lm15=0,lm30=0,lh1=0,lh4=0,ld1=0;
  datetime t=iTime(_Symbol,tf,0);
  if(tf==PERIOD_M1){ if(t!=lm1){ lm1=t; return true;} return false; }
  if(tf==PERIOD_M5){ if(t!=lm5){ lm5=t; return true;} return false; }
  if(tf==PERIOD_M15){ if(t!=lm15){ lm15=t; return true;} return false; }
  if(tf==PERIOD_M30){ if(t!=lm30){ lm30=t; return true;} return false; }
  if(tf==PERIOD_H1){ if(t!=lh1){ lh1=t; return true;} return false; }
  if(tf==PERIOD_H4){ if(t!=lh4){ lh4=t; return true;} return false; }
  if(t!=ld1){ ld1=t; return true;} return false;
}

bool StrongBody(ENUM_TIMEFRAMES tf,int sh,int minPct){
  double o=iOpen(_Symbol,tf,sh), c=iClose(_Symbol,tf,sh), h=iHigh(_Symbol,tf,sh), l=iLow(_Symbol,tf,sh);
  double rng=MathMax(1e-6, h-l), body=MathAbs(c-o);
  return (100.0*body/rng)>=minPct;
}

double ATRnow(int h){ double v; if(!Copy1(h,v,0)) return 0.0; return v; }

//-------------------------- Seasonality ----------------------------//
double wHour[24], wWeek[7]; bool seas_ready=false;

void BuildSeason(){
  ArrayInitialize(wHour,1.0); ArrayInitialize(wWeek,1.0);
  if(!InpUseSeasonality){ seas_ready=true; return; }
  int barsAvail=Bars(_Symbol,InpEntryTF);
  int N=MathMin(InpSeasonBarsBack, barsAvail-InpMinBars);
  if(N<=0){ seas_ready=true; return; }

  double sumH[24]; int cntH[24]; ArrayInitialize(sumH,0); ArrayInitialize(cntH,0);
  double sumW[7];  int cntW[7];  ArrayInitialize(sumW,0);  ArrayInitialize(cntW,0);

  for(int i=1;i<=N;i++){
    datetime t=iTime(_Symbol,InpEntryTF,i);
    int hr=HourOf(t), wd=WdayOf(t);
    double r=iClose(_Symbol,InpEntryTF,i)-iOpen(_Symbol,InpEntryTF,i);
    if(hr>=0&&hr<24){ sumH[hr]+=r; cntH[hr]++; }
    if(wd>=1&&wd<=5){ sumW[wd]+=r; cntW[wd]++; }
  }
  double meanH=0; int nH=0;
  for(int h=0;h<24;h++){ if(cntH[h]>0){ wHour[h]=sumH[h]/cntH[h]; meanH+=wHour[h]; nH++; } else wHour[h]=1.0; }
  meanH=(nH>0? meanH/nH : 0.0);
  for(int h2=0;h2<24;h2++){ if(cntH[h2]>0 && meanH!=0.0) wHour[h2]/=meanH; else wHour[h2]=1.0; }

  double meanW=0; int nW=0;
  for(int d=1;d<=5;d++){ if(cntW[d]>0){ wWeek[d]=sumW[d]/cntW[d]; meanW+=wWeek[d]; nW++; } else wWeek[d]=1.0; }
  meanW=(nW>0? meanW/nW : 0.0);
  for(int d2=1;d2<=5;d2++){ if(cntW[d2]>0 && meanW!=0.0) wWeek[d2]/=meanW; else wWeek[d2]=1.0; }

  seas_ready=true;
}

double SeasonW(datetime t){
  if(!seas_ready) BuildSeason();
  if(!InpUseSeasonality) return 1.0;
  int hr=HourOf(t), wd=WdayOf(t);
  double wh=(hr>=0&&hr<24? wHour[hr]:1.0);
  double ww=((wd>=1&&wd<=5)? wWeek[wd]:1.0);
  return wh*ww;
}

//-------------------------- Bias / Regime --------------------------//
enum Bias { BIAS_NONE=0, BIAS_BUY=1, BIAS_SELL=-1 };

Bias TrendBias(){
  if(Bars(_Symbol,InpTrendTF)<InpMinBars) return BIAS_NONE;
  double ma0,ma1; if(!Copy1(hEMA_trend,ma0,0) || !Copy1(hEMA_trend,ma1,1)) return BIAS_NONE;
  double px=iClose(_Symbol,InpTrendTF,0);
  bool up=(ma0>ma1), dn=(ma0<ma1);
  if(InpRequireSlope){
    if(px>ma0 && up) return BIAS_BUY;
    if(px<ma0 && dn) return BIAS_SELL;
  }else{
    if(px>ma0) return BIAS_BUY;
    if(px<ma0) return BIAS_SELL;
  }
  return BIAS_NONE;
}

bool TrendRegime(){ double adx; if(!Copy1(hADX, adx, 1)) return false; return (adx>=InpADX_TrendThresh); }

//========================== ENSEMBLE SIGNALS =======================//
struct BRState { int dir; double level; int setBar; } br = {0,0.0,-1};

double SignalBreakout(){
  int idxH=iHighest(_Symbol,InpEntryTF,MODE_HIGH,InpDonchLookback,1);
  int idxL=iLowest (_Symbol,InpEntryTF,MODE_LOW ,InpDonchLookback,1);
  if(idxH==-1||idxL==-1) return 0.0;
  double Lh=iHigh(_Symbol,InpEntryTF,idxH), Ll=iLow(_Symbol,InpEntryTF,idxL);
  double c1=iClose(_Symbol,InpEntryTF,1);

  if(br.dir==0){
    if(c1 > Lh + Pts(InpBreakBufPt)){
      int touches=0; for(int i=2;i<=InpDonchLookback;i++){ if(MathAbs(iHigh(_Symbol,InpEntryTF,i)-Lh)<=Pts((int)InpRetestBufPt)) touches++; }
      if(touches>=InpMinTouches){ br.dir=1; br.level=Lh; br.setBar=Bars(_Symbol,InpEntryTF); }
    }
    if(c1 < Ll - Pts(InpBreakBufPt)){
      int touches=0; for(int j=2;j<=InpDonchLookback;j++){ if(MathAbs(iLow(_Symbol,InpEntryTF,j)-Ll)<=Pts((int)InpRetestBufPt)) touches++; }
      if(touches>=InpMinTouches){ br.dir=-1; br.level=Ll; br.setBar=Bars(_Symbol,InpEntryTF); }
    }
    return 0.0;
  }

  bool expired=(Bars(_Symbol,InpEntryTF)-br.setBar)>InpMaxBarsRetest;
  if(expired){ br.dir=0; return 0.0; }

  double h0=iHigh(_Symbol,InpEntryTF,0), l0=iLow(_Symbol,InpEntryTF,0);
  bool touched = (br.dir==1? (l0<=br.level+Pts((int)InpRetestBufPt))
                           : (h0>=br.level-Pts((int)InpRetestBufPt)));
  if(touched && StrongBody(InpEntryTF,1,InpConfirmBodyPct)){
    int d=br.dir; br.dir=0; return (d==1? 1.0:-1.0);
  }
  return 0.0;
}

double SignalPullback(){
  double f0,s0; if(!Copy1(hEMA_fast,f0,0) || !Copy1(hEMA_slow,s0,0)) return 0.0;
  double c0=iClose(_Symbol,InpEntryTF,0), c1=iClose(_Symbol,InpEntryTF,1);
  bool nearBuy  = (MathAbs(c0 - f0) <= Pts((int)InpPullbackTolPt)) && (c0>c1);
  bool nearSell = (MathAbs(c0 - f0) <= Pts((int)InpPullbackTolPt)) && (c0<c1);
  if(nearBuy && f0>s0)  return 1.0;
  if(nearSell && f0<s0) return -1.0;
  return 0.0;
}

//========================= SIZING & PROTECT ========================//
struct SLTP { double entry, sl, tp1; double sl_points; };
bool CalcSLTP(bool isBuy, SLTP &o){
  if(!GetTick()) return false;
  int slpts = InpFixedSL_Points;
  if(InpUseATRforSL){
    double a=ATRnow(hATR_entry);
    slpts = MathMax(1,(int)MathRound((a*InpATR_Mult)/_Point));
  }
  double entry = isBuy? last_tick.ask:last_tick.bid;
  double sl    = isBuy? entry - Pts(slpts) : entry + Pts(slpts);
  double tp1   = isBuy? entry + Pts((int)(slpts*InpTP1_RR)) : entry - Pts((int)(slpts*InpTP1_RR));
  o.entry=entry; o.sl=sl; o.tp1=tp1; o.sl_points=slpts; return true;
}

double CalcLots(double sl_points, double riskScale=1.0){
  double bal=AccountInfoDouble(ACCOUNT_BALANCE);
  double risk=MathMax(0.0001, (InpRiskPercent*riskScale)/100.0)*bal;
  double tv=SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_VALUE);
  double ts=SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_SIZE);
  if(tv<=0||ts<=0) return 0.0;
  double money_per_point_1lot = tv*(_Point/ts);
  double lot = risk/(sl_points*money_per_point_1lot);
  double minlot=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
  double maxlot=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MAX);
  double step=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP); if(step<=0) step=0.01;
  lot=MathMax(minlot, MathMin(maxlot, MathFloor(lot/step)*step));
  return lot;
}

bool HasOpen(int &dir,double &vol,ulong &ticket){
  dir=0; vol=0; ticket=0;
  for(int i=0;i<PositionsTotal();i++){
    ulong t=PositionGetTicket(i); if(!PositionSelectByTicket(t)) continue;
    if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
    if(PositionGetInteger(POSITION_MAGIC)!=(long)InpMagic) continue;
    int type=(int)PositionGetInteger(POSITION_TYPE);
    dir=(type==POSITION_TYPE_BUY?1:-1); vol=PositionGetDouble(POSITION_VOLUME); ticket=t; return true;
  }
  return false;
}

int OpenPositionsAll(){ int n=0; for(int i=0;i<PositionsTotal();i++) n++; return n; }

//============================ TREND EXIT ===========================//
// Returns votes (0..4) for closing remainder in profit
double TrendExitVotes(int dir){
  double votes=0.0;

  // 1) Close beyond fast EMA N bars against position
  bool emaClose=false;
  for(int k=0;k<InpExit_EMA_Bars;k++){
    double c=iClose(_Symbol,InpEntryTF,k), f; if(!Copy1(hEMA_fast,f,k)) continue;
    if(dir==1 && c < f) { emaClose=true; break; }
    if(dir==-1 && c > f){ emaClose=true; break; }
  }
  if(emaClose) votes+=1.0;

  // 2) 20 vs 50 cross against direction
  if(InpExit_EMA_Cross){
    double f0,s0,f1,s1;
    if(Copy1(hEMA_fast,f0,0) && Copy1(hEMA_slow,s0,0) && Copy1(hEMA_fast,f1,1) && Copy1(hEMA_slow,s1,1)){
      bool crossDn = (f1>=s1 && f0<s0); // against buy
      bool crossUp = (f1<=s1 && f0>s0); // against sell
      if( (dir==1 && crossDn) || (dir==-1 && crossUp) ) votes+=1.0;
    }
  }

  // 3) ADX drops below trend threshold (loss of momentum)
  if(InpExit_ADX_Drop){
    double adx; if(Copy1(hADX,adx,1) && adx<InpADX_TrendThresh) votes+=1.0;
  }

  // 4) RSI midline flip against direction
  if(InpExit_RSI_MidFlip){
    double r0; if(Copy1(hRSI,r0,1)){
      if(dir==1 && r0<50.0) votes+=1.0;
      if(dir==-1 && r0>50.0) votes+=1.0;
    }
  }
  return votes;
}

//============================== MANAGE =============================//
void ManageOpen(){
  int dir; double vol; ulong tk; if(!HasOpen(dir,vol,tk)) return;

  // time-stop
  datetime open_time = (datetime)PositionGetInteger(POSITION_TIME);
  int barsSinceOpen = iBarShift(_Symbol, InpEntryTF, open_time) - iBarShift(_Symbol, InpEntryTF, TimeCurrent());
  if(barsSinceOpen < 0) barsSinceOpen = -barsSinceOpen;

  double entry=PositionGetDouble(POSITION_PRICE_OPEN);
  double sl   =PositionGetDouble(POSITION_SL);
  double cur  =(dir==1? (GetTick()?last_tick.bid:SymbolInfoDouble(_Symbol,SYMBOL_BID))
                     : (GetTick()?last_tick.ask:SymbolInfoDouble(_Symbol,SYMBOL_ASK)));
  double slpts=MathMax(1.0, MathAbs((entry-sl)/_Point));
  double tp1  =(dir==1? entry + Pts((int)(slpts*InpTP1_RR)) : entry - Pts((int)(slpts*InpTP1_RR)));

  // 1) Partial + BE at TP1
  if(InpPartialCloseFrac>0.0){
    bool hit=(dir==1? cur>=tp1:cur<=tp1);
    if(hit && vol>SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN)){
      double minv=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
      double step=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP); if(step<=0) step=0.01;
      double closeVol=MathMax(minv, NormalizeDouble(vol*InpPartialCloseFrac/step,0)*step);
      if(closeVol<vol){
        if(Trade.PositionClosePartial(tk, closeVol)){
          if(InpMoveSLtoBE){
            double be = entry + (dir==1?1:-1)*2*_Point;
            Trade.PositionModify(tk, be, 0.0);
          }
        }
      }
    }
  }

  // 2) ATR trailing for remainder
  if(InpEnableATRTrail){
    double a; if(!Copy1(hATR_trail,a,0)) return;
    double nd=InpATR_Trail_Mult*a;
    double newSL=(dir==1? cur-nd : cur+nd);
    if(dir==1 && newSL>sl) Trade.PositionModify(tk,newSL,0.0);
    if(dir==-1 && newSL<sl) Trade.PositionModify(tk,newSL,0.0);
  }

  // 3) Trend-Exit votes (close remainder in profit if confirmed)
  double votes = TrendExitVotes(dir);
  if(votes >= InpExitVoteNeed){
    Trade.PositionClose(tk);

    // Flip (optional) — open reverse with scaled risk
    if(InpEnableFlip){
      if(Bars(_Symbol,InpEntryTF) - lastFlipBar >= InpFlipCooldownBars){
        int want = (dir==1? -1: 1);
        SLTP s; if(!CalcSLTP(want==1, s)) return;
        double lots = CalcLots(s.sl_points, InpFlipRiskScale); if(lots<=0) return;
        Trade.SetExpertMagicNumber(InpMagic);
        Trade.SetDeviationInPoints(10);
        bool ok=false;
        if(want==1) ok=Trade.Buy(lots,NULL,0.0,s.sl,0.0,"V4.1 Flip Buy");
        else        ok=Trade.Sell(lots,NULL,0.0,s.sl,0.0,"V4.1 Flip Sell");
        if(ok) lastFlipBar = Bars(_Symbol,InpEntryTF);
      }
    }
  }

  // 4) time-stop (safety)
  if(barsSinceOpen >= InpTimeStopBars){
    Trade.PositionClose(tk);
  }
}

//============================ RISK GUARD ===========================//
bool DayChanged(){
   MqlDateTime dt; TimeToStruct(TimeCurrent(), dt);
   dt.hour = 0; dt.min = 0; dt.sec = 0;
   datetime start = StructToTime(dt);
   if(day_anchor == 0){ day_anchor = start; return true; }
   if(start != day_anchor){ day_anchor = start; return true; }
   return false;
}

bool RiskGuardAllows(){
  if(DayChanged())
     day_start_equity = AccountInfoDouble(ACCOUNT_EQUITY);
  if(day_start_equity <= 0.0)
     day_start_equity = AccountInfoDouble(ACCOUNT_EQUITY);

  double eq=AccountInfoDouble(ACCOUNT_EQUITY);
  if(eq <= day_start_equity*(1.0 - InpDailyLossCapPct/100.0)) return false;
  if(OpenPositionsAll() >= InpMaxOpenPositions) return false;
  if(SpreadPts() > InpMaxSpreadPoints) return false;
  return true;
}

//=============================== CORE ==============================//
string BiasStr(Bias b){ if(b==BIAS_BUY) return "BUY"; if(b==BIAS_SELL) return "SELL"; return "NONE"; }

void TryTrade(){
  if(!RiskGuardAllows()) return;
  if(InpOnePerSymbol){ int d; double v; ulong t; if(HasOpen(d,v,t)) return; }
  if(!IsNewBar(InpEntryTF)) return;

  Bias b = TrendBias(); if(b==BIAS_NONE) return;
  if(!TrendRegime()) return; // зөвхөн тренд үед орно

  // Seasonality
  datetime now=iTime(_Symbol,InpEntryTF,0);
  if(SeasonW(now) < InpMinSeasonWeight) return;

  // Ensemble (trend only)
  double s_break = SignalBreakout(); // +1/-1/0
  double s_pull  = SignalPullback(); // +1/-1/0

  double dirScore = W_Breakout*s_break + W_Pullback*s_pull;

  // Align with higher TF bias
  if(b==BIAS_BUY && dirScore < 0) dirScore = 0.0;
  if(b==BIAS_SELL && dirScore > 0) dirScore = 0.0;

  double score = MathAbs(dirScore);
  int    want  = (dirScore>0? 1 : (dirScore<0? -1:0));
  if(score < InpEnterScore || want==0) return;

  SLTP s; if(!CalcSLTP(want==1, s)) return;
  double lots = CalcLots(s.sl_points); if(lots<=0) return;

  Trade.SetExpertMagicNumber(InpMagic);
  Trade.SetDeviationInPoints(10);
  bool ok=false;
  if(want==1) ok=Trade.Buy(lots,NULL,0.0,s.sl,0.0,"V4.1 Buy");
  else        ok=Trade.Sell(lots,NULL,0.0,s.sl,0.0,"V4.1 Sell");
  if(ok) PrintFormat("[V4.1] %s Score=%.2f Bias=%s SLpts=%g Lots=%.2f", (want==1?"BUY":"SELL"), score, BiasStr(b), s.sl_points, lots);
}

void HUD(){
  Bias b=TrendBias();
  Comment(StringFormat("V4.1 TrendRide | Bias:%s | Spread:%.0fpt | DayEQ:%.0f→%.0f cap:%.1f%%",
           BiasStr(b), SpreadPts(), day_start_equity, AccountInfoDouble(ACCOUNT_EQUITY), InpDailyLossCapPct));
}

//=============================== EVENTS ============================//
int OnInit(){
  hATR_entry = iATR(_Symbol, InpEntryTF,  InpATR_Period);
  hATR_trail = iATR(_Symbol, InpEntryTF,  InpATR_Trail_Period);
  hEMA_fast  = iMA (_Symbol, InpEntryTF,  InpFastEMA_Period, 0, MODE_EMA, PRICE_CLOSE);
  hEMA_slow  = iMA (_Symbol, InpEntryTF,  InpSlowEMA_Period, 0, MODE_EMA, PRICE_CLOSE);
  hEMA_trend = iMA (_Symbol, InpTrendTF,  InpTrendMAPeriod,  0, MODE_EMA, PRICE_CLOSE);
  hADX       = iADX(_Symbol, InpEntryTF,  InpADX_Period);
  hRSI       = iRSI(_Symbol, InpEntryTF,  InpRSI_Period, PRICE_CLOSE);

  if(hATR_entry==INVALID_HANDLE || hATR_trail==INVALID_HANDLE ||
     hEMA_fast==INVALID_HANDLE || hEMA_slow==INVALID_HANDLE ||
     hEMA_trend==INVALID_HANDLE|| hADX==INVALID_HANDLE || hRSI==INVALID_HANDLE){
    Print("[V4.1] Indicator init failed"); return INIT_FAILED;
  }
  BuildSeason();
  day_anchor=0; day_start_equity=AccountInfoDouble(ACCOUNT_EQUITY);
  Print("[V4.1] Initialized.");
  return INIT_SUCCEEDED;
}

void OnDeinit(const int reason){ Comment(""); }
void OnTick(){ HUD(); ManageOpen(); TryTrade(); }

//+------------------------------------------------------------------+
//|                                         SuperBot_V4_Aegis.mq5    |
//| Regime + Ensemble (Breakout/Retest, Pullback-EMA, MeanRevert)    |
//| + RiskGuard (daily loss cap, time-stop, exposure)                 |
//| BABA × Hypatia (1117)                                            |
//+------------------------------------------------------------------+
#property strict
#property version   "4.0"
#property copyright "1117 / BABA & Hypatia"

#include <Trade/Trade.mqh>
CTrade Trade;

//============================== INPUTS =============================//
// --- General ---
input ulong  InpMagic             = 11170004;
input bool   InpOnePerSymbol      = true;
input ENUM_TIMEFRAMES InpEntryTF  = PERIOD_M5;

// --- Trend/Regime ---
input ENUM_TIMEFRAMES InpTrendTF  = PERIOD_H1;
input int    InpTrendMAPeriod     = 200;      // EMA(H1)
input bool   InpRequireSlope      = true;     // price on correct side + slope
input int    InpADX_Period        = 14;       // ADX for regime
input double InpADX_TrendThresh   = 18.0;     // >= → trend, < → range

// --- Ensemble Weights (0..1) ---
input double W_Breakout           = 0.50;
input double W_Pullback           = 0.30;
input double W_MeanRevert         = 0.20;
input double InpEnterScore        = 0.70;     // need >= to enter

// --- Breakout/Retest ---
input int    InpDonchLookback     = 30;
input double InpBreakBufPt        = 10;       // close beyond level
input double InpRetestBufPt       = 15;       // tolerance on retest
input int    InpMinTouches        = 2;
input int    InpMaxBarsRetest     = 8;        // bars to allow retest
input int    InpConfirmBodyPct    = 55;       // strong body confirmation

// --- Pullback-EMA (trend-follow) ---
input int    InpFastEMA_Period    = 20;       // on Entry TF
input int    InpSlowEMA_Period    = 50;
input double InpPullbackTolPt     = 20;       // distance to fastEMA

// --- Mean Reversion (range regime) ---
input int    InpRSI_Period        = 14;
input int    InpRSI_BuyLevel      = 30;
input int    InpRSI_SellLevel     = 70;

// --- Seasonality gate ---
input bool   InpUseSeasonality    = true;
input double InpMinSeasonWeight   = 0.90;
input int    InpSeasonBarsBack    = 10000;

// --- Risk / Sizing ---
input double InpRiskPercent       = 1.0;      // per trade of balance
input bool   InpUseATRforSL       = true;
input int    InpATR_Period        = 14;
input double InpATR_Mult          = 2.0;
input int    InpFixedSL_Points    = 300;      // if ATR=false
input double InpTP1_RR            = 1.0;      // TP1 = 1R
input double InpPartialCloseFrac  = 0.50;     // 50% at TP1
input bool   InpMoveSLtoBE        = true;
input bool   InpEnableATRTrail    = true;
input int    InpATR_Trail_Period  = 14;
input double InpATR_Trail_Mult    = 2.5;

// --- RiskGuard ---
input double InpDailyLossCapPct   = 2.0;      // stop trading if equity down ≥ X% from day start
input int    InpMaxOpenPositions  = 3;        // across account
input int    InpTimeStopBars      = 30;       // close if not hit 1R within N bars
input double InpMaxSpreadPoints   = 50;

// --- Hygiene ---
input int    InpMinBars           = 500;

//=========================== STATE/HANDLES ==========================//
int   hATR_entry = INVALID_HANDLE, hATR_trail = INVALID_HANDLE;
int   hEMA_fast  = INVALID_HANDLE, hEMA_slow  = INVALID_HANDLE;
int   hEMA_trend = INVALID_HANDLE, hADX       = INVALID_HANDLE;
int   hRSI       = INVALID_HANDLE;

struct Pending { int dir; double level; int setBar; };
Pending pend = {0,0.0,-1};

MqlTick last_tick;
datetime day_anchor = 0;
double   day_start_equity = 0.0;

//=============================== UTILS =============================//
double Pts(double p){ return p*_Point; }
bool   GetTick(){ return SymbolInfoTick(_Symbol,last_tick); }
double SpreadPts(){ double s=(double)SymbolInfoInteger(_Symbol,SYMBOL_SPREAD); if(s<=0 && GetTick()) s=(last_tick.ask-last_tick.bid)/_Point; return s; }
bool   Copy1(int h, double &v, int sh=0){ double b[]; if(CopyBuffer(h,0,sh,2,b)!=2) return false; v=b[0]; return true; }

int HourOf(datetime t){ MqlDateTime dt; TimeToStruct(t,dt); return (int)dt.hour; }
int WdayOf(datetime t){ MqlDateTime dt; TimeToStruct(t,dt); return (int)dt.day_of_week; }

bool IsNewBar(ENUM_TIMEFRAMES tf){
  static datetime lm1=0,lm5=0,lm15=0,lm30=0,lh1=0,lh4=0,ld1=0;
  datetime t=iTime(_Symbol,tf,0);
  if(tf==PERIOD_M1){ if(t!=lm1){ lm1=t; return true;} return false; }
  if(tf==PERIOD_M5){ if(t!=lm5){ lm5=t; return true;} return false; }
  if(tf==PERIOD_M15){ if(t!=lm15){ lm15=t; return true;} return false; }
  if(tf==PERIOD_M30){ if(t!=lm30){ lm30=t; return true;} return false; }
  if(tf==PERIOD_H1){ if(t!=lh1){ lh1=t; return true;} return false; }
  if(tf==PERIOD_H4){ if(t!=lh4){ lh4=t; return true;} return false; }
  if(t!=ld1){ ld1=t; return true;} return false;
}

bool StrongBody(ENUM_TIMEFRAMES tf,int sh,int minPct){
  double o=iOpen(_Symbol,tf,sh), c=iClose(_Symbol,tf,sh), h=iHigh(_Symbol,tf,sh), l=iLow(_Symbol,tf,sh);
  double rng=MathMax(1e-6, h-l), body=MathAbs(c-o);
  return (100.0*body/rng)>=minPct;
}

double ATRnow(int h){ double v; if(!Copy1(h,v,0)) return 0.0; return v; }

//-------------------------- Seasonality ----------------------------//
double wHour[24], wWeek[7]; bool seas_ready=false;

void BuildSeason(){
  ArrayInitialize(wHour,1.0); ArrayInitialize(wWeek,1.0);
  if(!InpUseSeasonality){ seas_ready=true; return; }
  int barsAvail=Bars(_Symbol,InpEntryTF);
  int N=MathMin(InpSeasonBarsBack, barsAvail-InpMinBars);
  if(N<=0){ seas_ready=true; return; }

  double sumH[24]; int cntH[24]; ArrayInitialize(sumH,0); ArrayInitialize(cntH,0);
  double sumW[7];  int cntW[7];  ArrayInitialize(sumW,0);  ArrayInitialize(cntW,0);

  for(int i=1;i<=N;i++){
    datetime t=iTime(_Symbol,InpEntryTF,i);
    int hr=HourOf(t), wd=WdayOf(t);
    double r=iClose(_Symbol,InpEntryTF,i)-iOpen(_Symbol,InpEntryTF,i);
    if(hr>=0&&hr<24){ sumH[hr]+=r; cntH[hr]++; }
    if(wd>=1&&wd<=5){ sumW[wd]+=r; cntW[wd]++; }
  }
  double meanH=0; int nH=0;
  for(int h=0;h<24;h++){ if(cntH[h]>0){ wHour[h]=sumH[h]/cntH[h]; meanH+=wHour[h]; nH++; } else wHour[h]=1.0; }
  meanH=(nH>0? meanH/nH : 0.0);
  for(int h2=0;h2<24;h2++){ if(cntH[h2]>0 && meanH!=0.0) wHour[h2]/=meanH; else wHour[h2]=1.0; }

  double meanW=0; int nW=0;
  for(int d=1;d<=5;d++){ if(cntW[d]>0){ wWeek[d]=sumW[d]/cntW[d]; meanW+=wWeek[d]; nW++; } else wWeek[d]=1.0; }
  meanW=(nW>0? meanW/nW : 0.0);
  for(int d2=1;d2<=5;d2++){ if(cntW[d2]>0 && meanW!=0.0) wWeek[d2]/=meanW; else wWeek[d2]=1.0; }

  seas_ready=true;
}

double SeasonW(datetime t){
  if(!seas_ready) BuildSeason();
  if(!InpUseSeasonality) return 1.0;
  int hr=HourOf(t), wd=WdayOf(t);
  double wh=(hr>=0&&hr<24? wHour[hr]:1.0);
  double ww=((wd>=1&&wd<=5)? wWeek[wd]:1.0);
  return wh*ww;
}

//-------------------------- Regime --------------------------------//
enum Regime { REG_NONE=0, REG_TREND=1, REG_RANGE=2 };
enum Bias   { BIAS_NONE=0, BIAS_BUY=1, BIAS_SELL=-1 };

Bias TrendBias(){
  if(Bars(_Symbol,InpTrendTF)<InpMinBars) return BIAS_NONE;
  double ma0,ma1; if(!Copy1(hEMA_trend,ma0,0) || !Copy1(hEMA_trend,ma1,1)) return BIAS_NONE;
  double px=iClose(_Symbol,InpTrendTF,0);
  bool up=(ma0>ma1), dn=(ma0<ma1);
  if(InpRequireSlope){
    if(px>ma0 && up) return BIAS_BUY;
    if(px<ma0 && dn) return BIAS_SELL;
  }else{
    if(px>ma0) return BIAS_BUY;
    if(px<ma0) return BIAS_SELL;
  }
  return BIAS_NONE;
}

Regime MarketRegime(){
  double adx; if(!Copy1(hADX, adx, 1)) return REG_NONE; // use closed bar
  if(adx>=InpADX_TrendThresh) return REG_TREND;
  return REG_RANGE;
}

//========================== ENSEMBLE SIGNALS =======================//
// Breakout/Retest engine (sets 'pend'; returns score 0..1 when final confirms)
double SignalBreakout(){
  // Compute Donchian
  int idxH=iHighest(_Symbol,InpEntryTF,MODE_HIGH,InpDonchLookback,1);
  int idxL=iLowest (_Symbol,InpEntryTF,MODE_LOW ,InpDonchLookback,1);
  if(idxH==-1||idxL==-1) return 0.0;
  double Lh=iHigh(_Symbol,InpEntryTF,idxH), Ll=iLow(_Symbol,InpEntryTF,idxL);
  double c1=iClose(_Symbol,InpEntryTF,1);

  // set pending after break (closed bar)
  if(pend.dir==0){
    if(c1 > Lh + Pts(InpBreakBufPt)){
      int touches=0; for(int i=2;i<=InpDonchLookback;i++){ if(MathAbs(iHigh(_Symbol,InpEntryTF,i)-Lh)<=Pts((int)InpRetestBufPt)) touches++; }
      if(touches>=InpMinTouches){ pend.dir=1; pend.level=Lh; pend.setBar=Bars(_Symbol,InpEntryTF); }
    }
    if(c1 < Ll - Pts(InpBreakBufPt)){
      int touches=0; for(int j=2;j<=InpDonchLookback;j++){ if(MathAbs(iLow(_Symbol,InpEntryTF,j)-Ll)<=Pts((int)InpRetestBufPt)) touches++; }
      if(touches>=InpMinTouches){ pend.dir=-1; pend.level=Ll; pend.setBar=Bars(_Symbol,InpEntryTF); }
    }
    return 0.0;
  }

  // wait retest + strong body (previous bar)
  bool expired=(Bars(_Symbol,InpEntryTF)-pend.setBar)>InpMaxBarsRetest;
  if(expired){ pend.dir=0; return 0.0; }

  double h0=iHigh(_Symbol,InpEntryTF,0), l0=iLow(_Symbol,InpEntryTF,0);
  bool touched = (pend.dir==1? (l0<=pend.level+Pts((int)InpRetestBufPt))
                            : (h0>=pend.level-Pts((int)InpRetestBufPt)));
  if(touched && StrongBody(InpEntryTF,1,InpConfirmBodyPct)){
    int d=pend.dir; pend.dir=0; // consume
    return (d==1? 1.0 : -1.0);  // +1 buy, -1 sell
  }
  return 0.0;
}

// Pullback to EMA (trend regime) → score +1/-1
double SignalPullback(){
  double f0,s0; if(!Copy1(hEMA_fast,f0,0) || !Copy1(hEMA_slow,s0,0)) return 0.0;
  double c0=iClose(_Symbol,InpEntryTF,0), c1=iClose(_Symbol,InpEntryTF,1);
  // distance to fast EMA within tolerance, and bounce candle (c1->c0 direction)
  bool nearBuy  = (MathAbs(c0 - f0) <= Pts((int)InpPullbackTolPt)) && (c0>c1);
  bool nearSell = (MathAbs(c0 - f0) <= Pts((int)InpPullbackTolPt)) && (c0<c1);
  if(nearBuy && f0>s0)  return 1.0;
  if(nearSell && f0<s0) return -1.0;
  return 0.0;
}

// Mean reversion (range regime) → RSI oversold/overbought snap
double SignalMeanRevert(){
  double r1; if(!Copy1(hRSI,r1,1)) return 0.0;
  double c0=iClose(_Symbol,InpEntryTF,0), c1=iClose(_Symbol,InpEntryTF,1);
  if(r1<=InpRSI_BuyLevel && c0>c1)  return 1.0;   // bounce up
  if(r1>=InpRSI_SellLevel && c0<c1) return -1.0;  // bounce down
  return 0.0;
}

//========================= SIZING & PROTECT ========================//
struct SLTP { double entry, sl, tp1; double sl_points; };
bool CalcSLTP(bool isBuy, SLTP &o){
  if(!GetTick()) return false;
  int slpts = InpFixedSL_Points;
  if(InpUseATRforSL){
    double a=ATRnow(hATR_entry);
    slpts = MathMax(1,(int)MathRound((a*InpATR_Mult)/_Point));
  }
  double entry = isBuy? last_tick.ask:last_tick.bid;
  double sl    = isBuy? entry - Pts(slpts) : entry + Pts(slpts);
  double tp1   = isBuy? entry + Pts((int)(slpts*InpTP1_RR)) : entry - Pts((int)(slpts*InpTP1_RR));
  o.entry=entry; o.sl=sl; o.tp1=tp1; o.sl_points=slpts; return true;
}

double CalcLots(double sl_points){
  double bal=AccountInfoDouble(ACCOUNT_BALANCE);
  double risk=MathMax(0.0001, InpRiskPercent/100.0)*bal;
  double tv=SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_VALUE);
  double ts=SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_SIZE);
  if(tv<=0||ts<=0) return 0.0;
  double money_per_point_1lot = tv*(_Point/ts);
  double lot = risk/(sl_points*money_per_point_1lot);
  double minlot=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
  double maxlot=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MAX);
  double step=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP); if(step<=0) step=0.01;
  lot=MathMax(minlot, MathMin(maxlot, MathFloor(lot/step)*step));
  return lot;
}

bool HasOpen(int &dir,double &vol,ulong &ticket){
  dir=0; vol=0; ticket=0;
  for(int i=0;i<PositionsTotal();i++){
    ulong t=PositionGetTicket(i); if(!PositionSelectByTicket(t)) continue;
    if(PositionGetString(POSITION_SYMBOL)!=_Symbol) continue;
    if(PositionGetInteger(POSITION_MAGIC)!=(long)InpMagic) continue;
    int type=(int)PositionGetInteger(POSITION_TYPE);
    dir=(type==POSITION_TYPE_BUY?1:-1); vol=PositionGetDouble(POSITION_VOLUME); ticket=t; return true;
  }
  return false;
}

int OpenPositionsAll(){
  int n=0; for(int i=0;i<PositionsTotal();i++) n++;
  return n;
}

void ManageOpen(){
  int dir; double vol; ulong tk; if(!HasOpen(dir,vol,tk)) return;

  // time-stop
  datetime open_time = (datetime)PositionGetInteger(POSITION_TIME);
   int barsSinceOpen = iBarShift(_Symbol, InpEntryTF, open_time) - iBarShift(_Symbol, InpEntryTF, TimeCurrent());
   if(barsSinceOpen < 0) barsSinceOpen = -barsSinceOpen;

  double entry=PositionGetDouble(POSITION_PRICE_OPEN);
  double sl   =PositionGetDouble(POSITION_SL);
  double cur  =(dir==1? (GetTick()?last_tick.bid:SymbolInfoDouble(_Symbol,SYMBOL_BID)) :
                         (GetTick()?last_tick.ask:SymbolInfoDouble(_Symbol,SYMBOL_ASK)));
  double slpts=MathMax(1.0, MathAbs((entry-sl)/_Point));
  double tp1  =(dir==1? entry + Pts((int)(slpts*InpTP1_RR)) : entry - Pts((int)(slpts*InpTP1_RR)));

  // partial + BE
  if(InpPartialCloseFrac>0.0){
    bool hit=(dir==1? cur>=tp1:cur<=tp1);
    if(hit && vol>SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN)){
      double minv=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
      double step=SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP); if(step<=0) step=0.01;
      double closeVol=MathMax(minv, NormalizeDouble(vol*InpPartialCloseFrac/step,0)*step);
      if(closeVol<vol){
        if(Trade.PositionClosePartial(tk, closeVol)){
          if(InpMoveSLtoBE){
            double be = entry + (dir==1?1:-1)*2*_Point;
            Trade.PositionModify(tk, be, 0.0);
          }
        }
      }
    }
  }

  // ATR trailing
  if(InpEnableATRTrail){
    double a; if(!Copy1(hATR_trail,a,0)) return;
    double nd=InpATR_Trail_Mult*a;
    double newSL=(dir==1? cur-nd : cur+nd);
    if(dir==1 && newSL>sl) Trade.PositionModify(tk,newSL,0.0);
    if(dir==-1 && newSL<sl) Trade.PositionModify(tk,newSL,0.0);
  }

  // time-stop close
  if(barsSinceOpen >= InpTimeStopBars){
    Trade.PositionClose(tk);
  }
}

//============================ RISK GUARD ===========================//
bool DayChanged(){
   MqlDateTime dt;
   TimeToStruct(TimeCurrent(), dt);
   // тухайн өдрийн 00:00 болгож нормальчлах
   dt.hour = 0; dt.min = 0; dt.sec = 0;
   datetime start = StructToTime(dt);

   if(day_anchor == 0){        // эхний ажиллалт
      day_anchor = start;
      return true;
   }
   if(start != day_anchor){    // шинэ өдөр эхэлжээ
      day_anchor = start;
      return true;
   }
   return false;
}


bool RiskGuardAllows(){
  if(DayChanged())
   day_start_equity = AccountInfoDouble(ACCOUNT_EQUITY);
   if(day_start_equity <= 0.0)
   day_start_equity = AccountInfoDouble(ACCOUNT_EQUITY);

  // daily equity drawdown cap
  double eq=AccountInfoDouble(ACCOUNT_EQUITY);
  if(eq <= day_start_equity*(1.0 - InpDailyLossCapPct/100.0)) return false;

  // exposure
  if(OpenPositionsAll() >= InpMaxOpenPositions) return false;

  // spread
  if(SpreadPts() > InpMaxSpreadPoints) return false;

  return true;
}

//=============================== CORE ==============================//
string BiasStr(Bias b){ if(b==BIAS_BUY) return "BUY"; if(b==BIAS_SELL) return "SELL"; return "NONE"; }
string RegStr(Regime r){ if(r==REG_TREND) return "TREND"; if(r==REG_RANGE) return "RANGE"; return "NONE"; }

void TryTrade(){
  if(!RiskGuardAllows()) return;
  if(InpOnePerSymbol){ int d; double v; ulong t; if(HasOpen(d,v,t)) return; }
  if(!IsNewBar(InpEntryTF)) return;

  Bias b = TrendBias(); if(b==BIAS_NONE) return;
  Regime r = MarketRegime(); if(r==REG_NONE) return;

  // seasonality gate
  datetime now=iTime(_Symbol,InpEntryTF,0);
  double sw = SeasonW(now);
  if(sw < InpMinSeasonWeight) return;

  // ensemble
  double s_break = SignalBreakout(); // +1/-1/0
  double s_pull  = (r==REG_TREND ? SignalPullback() : 0.0);
  double s_mean  = (r==REG_RANGE ? SignalMeanRevert() : 0.0);

  // direction consistency with higher TF bias
  double dirScore = 0.0;
  dirScore += W_Breakout * s_break;
  dirScore += W_Pullback * s_pull;
  dirScore += W_MeanRevert * s_mean;

  // keep only signals that agree with bias
  if(b==BIAS_BUY && dirScore < 0) dirScore = 0.0;
  if(b==BIAS_SELL && dirScore > 0) dirScore = 0.0;

  double score = MathAbs(dirScore);       // 0..1
  int    want  = (dirScore>0? 1 : (dirScore<0? -1:0));

  if(score >= InpEnterScore && want!=0){
    SLTP s; if(!CalcSLTP(want==1, s)) return;
    double lots = CalcLots(s.sl_points); if(lots<=0) return;

    Trade.SetExpertMagicNumber(InpMagic);
    Trade.SetDeviationInPoints(10);
    bool ok=false;
    if(want==1) ok=Trade.Buy(lots,NULL,0.0,s.sl,0.0,"V4 Buy");
    else        ok=Trade.Sell(lots,NULL,0.0,s.sl,0.0,"V4 Sell");

    if(ok) PrintFormat("[V4] %s | Reg:%s Bias:%s Score:%.2f SW:%.2f SLpts:%g Lots:%.2f",
                       (want==1?"BUY":"SELL"), RegStr(r), BiasStr(b), score, sw, s.sl_points, lots);
  }
}

void HUD(){
  Bias b=TrendBias(); Regime r=MarketRegime();
  Comment(StringFormat("V4 Aegis | Reg:%s | Bias:%s | Spread:%.0fpt | DayEQ:%.0f→%.0f cap:%.1f%%",
           RegStr(r), BiasStr(b), SpreadPts(), day_start_equity, AccountInfoDouble(ACCOUNT_EQUITY), InpDailyLossCapPct));
}

//=============================== EVENTS ============================//
int OnInit(){
  hATR_entry = iATR(_Symbol, InpEntryTF,  InpATR_Period);
  hATR_trail = iATR(_Symbol, InpEntryTF,  InpATR_Trail_Period);
  hEMA_fast  = iMA (_Symbol, InpEntryTF,  InpFastEMA_Period, 0, MODE_EMA, PRICE_CLOSE);
  hEMA_slow  = iMA (_Symbol, InpEntryTF,  InpSlowEMA_Period, 0, MODE_EMA, PRICE_CLOSE);
  hEMA_trend = iMA (_Symbol, InpTrendTF,  InpTrendMAPeriod,  0, MODE_EMA, PRICE_CLOSE);
  hADX       = iADX(_Symbol, InpEntryTF,  InpADX_Period);
  hRSI       = iRSI(_Symbol, InpEntryTF,  InpRSI_Period, PRICE_CLOSE);

  if(hATR_entry==INVALID_HANDLE || hATR_trail==INVALID_HANDLE || hEMA_fast==INVALID_HANDLE ||
     hEMA_slow==INVALID_HANDLE || hEMA_trend==INVALID_HANDLE || hADX==INVALID_HANDLE || hRSI==INVALID_HANDLE){
    Print("[V4] Indicator init failed"); return INIT_FAILED;
  }
  BuildSeason();
  day_anchor=0; day_start_equity=AccountInfoDouble(ACCOUNT_EQUITY);
  Print("[V4] Initialized.");
  return INIT_SUCCEEDED;
}

void OnDeinit(const int reason){ Comment(""); }

void OnTick(){
  HUD();
  ManageOpen();
  TryTrade();
}

