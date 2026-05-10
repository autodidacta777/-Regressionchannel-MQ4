//+------------------------------------------------------------------+
//|  RegressionChannel.mq4                                           |
//|  Canal de regresión lineal dinámico con bandas de desviación     |
//|  Se recalcula en cada nueva vela automáticamente                 |
//+------------------------------------------------------------------+
#property copyright "Diseño propio"
#property version   "1.00"
#property strict
#property indicator_chart_window

//--- Número de buffers: línea central + banda superior + banda inferior
#property indicator_buffers 3
#property indicator_plots   3

//--- Línea central (regresión)
#property indicator_label1  "Regresión"
#property indicator_type1   DRAW_LINE
#property indicator_color1  clrDodgerBlue
#property indicator_style1  STYLE_SOLID
#property indicator_width1  2

//--- Banda superior
#property indicator_label2  "Banda Superior"
#property indicator_type2   DRAW_LINE
#property indicator_color2  clrOrangeRed
#property indicator_style2  STYLE_DOT
#property indicator_width2  1

//--- Banda inferior
#property indicator_label3  "Banda Inferior"
#property indicator_type3   DRAW_LINE
#property indicator_color3  clrOrangeRed
#property indicator_style3  STYLE_DOT
#property indicator_width3  1

//--- Parámetros externos
extern int    Period_Bars    = 100;   // Número de velas del canal
extern double StdDev_Multi   = 2.0;   // Multiplicador de desviación estándar
extern bool   UseClose       = true;  // true=Close | false=mediana (H+L)/2

//--- Buffers
double RegLine[];
double UpperBand[];
double LowerBand[];

//--- Control de recálculo
datetime lastBarTime = 0;

//+------------------------------------------------------------------+
//| Inicialización                                                   |
//+------------------------------------------------------------------+
int OnInit()
  {
   // Validar parámetros
   if(Period_Bars < 2)
     {
      Alert("Period_Bars debe ser >= 2");
      return(INIT_PARAMETERS_INCORRECT);
     }

   // Asociar buffers
   SetIndexBuffer(0, RegLine);
   SetIndexBuffer(1, UpperBand);
   SetIndexBuffer(2, LowerBand);

   // Etiquetas en el tooltip del cursor
   SetIndexLabel(0, "Regresión(" + IntegerToString(Period_Bars) + ")");
   SetIndexLabel(1, "Superior +" + DoubleToString(StdDev_Multi, 1) + "σ");
   SetIndexLabel(2, "Inferior -"  + DoubleToString(StdDev_Multi, 1) + "σ");

   // Inicializar buffers con valor vacío
   SetIndexEmptyValue(0, EMPTY_VALUE);
   SetIndexEmptyValue(1, EMPTY_VALUE);
   SetIndexEmptyValue(2, EMPTY_VALUE);

   // El indicador necesita al menos Period_Bars velas para dibujar
   SetIndexDrawBegin(0, Period_Bars - 1);
   SetIndexDrawBegin(1, Period_Bars - 1);
   SetIndexDrawBegin(2, Period_Bars - 1);

   IndicatorShortName("RegChannel(" + IntegerToString(Period_Bars) +
                      ", " + DoubleToString(StdDev_Multi, 1) + "σ)");
   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
//| Cálculo principal                                                |
//+------------------------------------------------------------------+
int OnCalculate(const int rates_total,
                const int prev_calculated,
                const datetime &time[],
                const double   &open[],
                const double   &high[],
                const double   &low[],
                const double   &close[],
                const long     &tick_volume[],
                const long     &volume[],
                const int      &spread[])
  {
   // Necesitamos al menos Period_Bars velas
   if(rates_total < Period_Bars) return(0);

   // Calcular desde qué barra empezar
   // En el primer cálculo recalculamos todo el histórico
   // En cálculos posteriores solo actualizamos la última barra
   int startBar = (prev_calculated == 0) ? Period_Bars - 1 : prev_calculated - 1;

   for(int bar = startBar; bar < rates_total; bar++)
     {
      // Índice de inicio de la ventana para esta barra
      int windowStart = bar - Period_Bars + 1;
      if(windowStart < 0) { windowStart = 0; }

      int n = bar - windowStart + 1;  // Número real de barras en la ventana

      // Calcular regresión lineal por mínimos cuadrados
      // y = a + b*x   donde x = 0,1,...,n-1  e y = precio[bar-x]
      double sumX  = 0, sumY  = 0, sumXY = 0, sumX2 = 0;

      for(int i = 0; i < n; i++)
        {
         int idx = bar - i;
         double price = UseClose ? close[idx] : (high[idx] + low[idx]) / 2.0;
         double x = (double)i;

         sumX  += x;
         sumY  += price;
         sumXY += x * price;
         sumX2 += x * x;
        }

      double denom = (double)n * sumX2 - sumX * sumX;
      if(MathAbs(denom) < 1e-10)
        {
         RegLine[bar]   = EMPTY_VALUE;
         UpperBand[bar] = EMPTY_VALUE;
         LowerBand[bar] = EMPTY_VALUE;
         continue;
        }

      // Coeficientes de la recta
      // Nota: x=0 corresponde a la barra más reciente (bar)
      //       x=n-1 corresponde a la barra más antigua (windowStart)
      double b = ((double)n * sumXY - sumX * sumY) / denom;
      double a = (sumY - b * sumX) / (double)n;

      // Valor de la regresión en x=0 (barra actual)
      double regValue = a;

      // Calcular desviación estándar de los residuos
      double sumResid2 = 0;
      for(int i = 0; i < n; i++)
        {
         int idx = bar - i;
         double price = UseClose ? close[idx] : (high[idx] + low[idx]) / 2.0;
         double predicted = a + b * (double)i;
         double resid = price - predicted;
         sumResid2 += resid * resid;
        }

      double stdDev = (n > 1) ? MathSqrt(sumResid2 / (double)(n - 1)) : 0.0;

      // Asignar valores a los buffers
      RegLine[bar]   = regValue;
      UpperBand[bar] = regValue + StdDev_Multi * stdDev;
      LowerBand[bar] = regValue - StdDev_Multi * stdDev;
     }

   return(rates_total);
  }

//+------------------------------------------------------------------+
