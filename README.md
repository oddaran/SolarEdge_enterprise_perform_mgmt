# SolarEdge_enterprise_perform_mgmt Analysis & Planning
The project creates the framework and business strategy for a renewable energy manufacturer and distributor, SolarEdge Technologies. I describe the renewables industry, company products, and growth, addressing a hypothetical business problem. Knowing that business intelligence aligns with and merges into organizational strategy, I conceptualize a framework and compose several business questions meant to achieve enterprise objectives. Using public reported data compiled from corporate reports, I answer the questions with statistic applications from SAS web-based environment. After discussing results and industry-relevant considerations, I endorse ways to make further research and analysis.

# SolarEdge strategic goals
	Grow revenue and gross margin through strength in its products.
	Drive sustainable energy by developing and maximizing power from inverter and photovoltaic systems.
	Advance smart energy in market segments (residential, commercial, utility).
	Be the preferred partner for industry installers, integrators, and market participants.
	Maintain social responsibility through international energy and electricity standards.

# Viewing and summarizing data set energydata
* Upload csv to SAS, with header columns appropriately, format year as 4 digits, converting numeric to dollars;
data energydata;
	infile '/home/u43067822/sasuser.v94/energydata_solaredge10k.csv' dlm=',' firstobs=2;
	input Year World_TW US_TW Megawatt_shipped Inverters Optimizers EPS Revenue RD Asset_financed Gross_profit;

proc datasets library=work nolist;
    modify energydata;
    format year yy4.
    	inverters optimizers comma12.0
		eps dollar12.2
		revenue rd asset_financed gross_profit dollar18.2;
run;
proc print data=energydata;
run;

# Code begins here for business questions and statistics applications using SAS Studio
# 1. Is SolarEdge beating the market growth for inverters shipped?
* Transpose to a long format;
proc transpose data=energydata out=long;
by Year;
var Inverters Optimizers Megawatt_shipped Revenue Gross_profit;
run;

* Sort each variable to be consistent;
proc sort data=long;
by _name_ Year;
run;

* Calculate lag and remove lag calculation column;
data segrowth (DROP=prev_val);
set long;
by _name_;
prev_val = lag(col1);
if not(first._name_) then growth = (col1 / prev_val - 1);
proc print data=segrowth;
run;

* Calculate mean growth rate;
proc means data=segrowth MEAN;
var growth;
by _name_;
TITLE 'Average Growth Rate';
run;

* Visualize the growth by sorted _name_;
proc sort data=WORK.SEGROWTH out=_LineChartTaskData;
	by _NAME_;
run;

proc sgplot data=_LineChartTaskData;
	by _NAME_;
	title height=14pt "SolarEdge Annual Growth Rate";
	footnote2 justify=left height=8pt "Using public 10-k annual reporting data";
	vline Year / response=growth lineattrs=(thickness=2 color=CXed8b40);
	xaxis label="Year";
	yaxis grid;
run;

ods graphics / reset;
title;
footnote2;

proc datasets library=WORK noprint;
	delete _LineChartTaskData;
	run;

# 2.	Is growth in gross profit correlated to growth of power capacity (MW) produced?
* Calculate lag and remove lag calculation column for Profit & MW capacity;
data segrowth3 (DROP=World_TW US_TW Inverters Optimizers EPS Revenue RD Asset_financed prev_val prev_val2);
set energydata;
by Year;
prev_val2 = lag(Megawatt_shipped);
if not(first.Megawatt_shipped) then mw_growth = (Megawatt_shipped / prev_val2 - 1);
prev_val = lag(Gross_profit);
if not(first.Gross_profit) then gp_growth = (Gross_profit / prev_val - 1);
proc print data=segrowth3;
run;

* Calculate mean growth rates for MW shipped, Profit;
proc means data=segrowth3 MEAN;
var gp_growth mw_growth;
TITLE 'Average Growth Rate';
run;

# 3.	Can SolarEdge maintain innovation in product research and development?
* Create new dataframe and calculate r&d as proportion of revenue;
data proportion (drop= World_TW US_TW Megawatt_shipped Inverters Optimizers EPS Asset_financed);
set energydata;
proprd = RD / Revenue;
run;
TITLE 'Annual Proportion of R&D to Revenues';
proc print data=proportion;

* Run the one-way frequencies;
proc freq data=proportion;
	tables Year / plots=(freqplot cumfreqplot);
	weight proprd;
run;

proc means data=proportion;
var proprd;
run;

# 4.	Is product power capacity (GW) a realistic indicator of revenue?
* Prepare time series into numeric data for lin reg;
Data energydata1(drop= World_TW US_TW Inverters Optimizers EPS RD Asset_financed);
 set energydata;
   time = Year - 2010;
proc print data=energydata1;
   var time Megawatt_shipped Revenue;
   title 'Prepared Time Series for Lin Regression';
run;

* Run lin regression on energydata1, using revenue as dependent var, intercept as megawatt_shipped, and independent var;
* as megawatt_shipped, alpha level 0.01;
* Note the lin reg equation, using slope & intercept is Y=233500(X) + 62018305. p-value <.0001;
proc reg data=WORK.ENERGYDATA1 alpha=0.01 plots(only)=(diagnostics residuals 
		fitplot observedbypredicted);
	model Revenue=Megawatt_shipped /;
	run;
quit;

# 5.  Can we forecast power capacities produced through 2025?
* Run forecasting, then modeling/forecasting using ARIMA, defined by random walk;
proc arima data=WORK.ENERGYDATA plots
    (only)=(series(corr crosscorr) residual(corr normal) 
		forecast(forecastonly)) out=work.movingaveragemw;
	identify var=Megawatt_shipped (1);
	estimate method=CLS;
	forecast lead=5 back=0 alpha=0.01;
	outlier;
	run;
quit;

# References
Carlin, W. (2022). Solar power generation [energy data explorer]. https://ourworldindata.org/explorers/energy? 
Faier, R. (2021 Nov). Presentation: solaredge technologies. https://investors.solaredge.com/static-files/c0b9b9ea-d7d0-4cfa-8018-b979b63dedfaReferences 
Matijssen, M. (2020). Powering the future of energy: sustainability report 2020. https://www.solaredge.com/sites/default/files/annual_sustainability_report.pdf 
Solaredge. (2021). Annual reports. https://investors.solaredge.com/financial-information/annual-reports 
