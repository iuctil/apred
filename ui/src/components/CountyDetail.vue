<template>
<div ref="scrolled-area" @scroll="handleScroll" v-if="detail">
    <div class="header">
        <div class="page">
            <div class="toolset">
                <CountySelecter @select="$router.replace('/county/'+$event)" placeholder="Search other County" style="width: 180px;"/>
                or <el-button @click="compare" style="margin-left: 10px;" >Compare Counties</el-button>
            </div>
            <h3 style="font-weight: normal;">
                <el-button type="primary" circle icon="el-icon-back" @click="goback()" class="back-button"/>
                &nbsp;
                &nbsp;
                <span class="important"><b>{{detail.county}}</b> county,</span>
                {{detail.state}}
            </h3>
            <br clear="both">
            <el-tabs v-model="tab" @tab-click="scroll2top">
                <el-tab-pane name="storms" label="Storm History"></el-tab-pane>
                <el-tab-pane name="disaster" :label="'FEMA Disaster Declarations ('+history.length+')'"></el-tab-pane>
                <el-tab-pane v-if="detail.bvis2" name="bvi" label="Business Sector Vulnerability"></el-tab-pane>
                <el-tab-pane name="resilience" label="Disaster Resilience"></el-tab-pane>
                <el-tab-pane v-if="$root.isEDA" name="cost" label="Potential Disaster Cost"></el-tab-pane>
                <el-tab-pane name="info" label="County Detail"></el-tab-pane>                
            </el-tabs>
        </div>
    </div>
    <div class="page" v-if="tab == 'info'">
        <br>
        <div class="overview">
            <p>This page presents information on measuring distress indicators, economic, and demographic data for <b>{{detail.county}} County, {{detail.state}}</b>.</p>
        </div>

        <el-row :gutter="20">
            <el-col :span="5">
                <h4>Population History<br>
                    <small><a href="https://www.census.gov/programs-surveys/popest.html" target="census">(Census Bureau)</a></small>
                </h4>
            </el-col>
            <el-col :span="19">
                <ExportablePlotly :data="demoGraphData" :layout="demoGraphLayout"/>
                <br>
                <br>
            </el-col>
        </el-row>

        <el-row :gutter="20">
            <el-col :span="5">
                <h4>Racial Makeup History<br>
                    <small><a href="https://www.census.gov/programs-surveys/popest.html" target="census">(Census Bureau)</a></small>
                </h4>
            </el-col>
            <el-col :span="19">
                <ExportablePlotly :data="orGraphData" :layout="orGraphLayout"/>
                <br>
                <br>
            </el-col>
        </el-row>

        <el-row :gutter="20">
            <el-col :span="5">
                <h4>
                    Unemployment Rate<br>
                    <small>(24-month avg. BLS)</small>
                </h4>

                <div>
                    <h5>US Average</h5>
                    <span class="primary" style="color: gray;"> {{unempRateUS|formatNumber("0.00")}}%</span>
                </div>

                <div>
                    <h5><b>{{detail.county}}</b> County</h5>
                    <span class="primary"> {{unempRateCounty|formatNumber("0.00")}}%</span>
                    <br>
                    <el-tag type="danger" size="mini" v-if="unempRateUS < unempRateCounty">Above US Rate</el-tag>
                </div>

                <div style="opacity: 0.6">
                    <h5>Report Date</h5>
                    {{new Date(unempRateDate).toLocaleDateString('en-US', {timeZone: 'UTC'})}}
                </div>

            </el-col>
            <el-col :span="19">
                <p v-if="$root.commonReady">
                    <ExportablePlotly :data="urGraphData" :layout="urGraphLayout"/>
                    <small>Calculated by taking the sum of unemployed persons for one geography for the previous 24 months divided by the sum of the labor force for that geography for the previous 24 months. Released monthly as part of the Local Area Unemployment Statistics (LAUS) program.</small>
                </p>

                <p>
                    <el-button round @click="showEmploymentHistory = !showEmploymentHistory" size="small">
                        <i class="el-icon-caret-right" v-if="!showEmploymentHistory"/> 
                        <i class="el-icon-caret-bottom" v-if="showEmploymentHistory"/> 
                        Show Employment History
                    </el-button>

                    <ExportablePlotly v-if="$root.commonReady && showEmploymentHistory" :data="uGraphData" :layout="uGraphLayout"/>
                </p>
                <br>
            </el-col>
        </el-row>

        <el-row :gutter="20">
            <el-col :span="5">
                <h4>Total Households</h4>
                <span class="primary">
                    {{detail.households|formatNumber}}
                </span>
                <br>
                <small><a href="https://www.census.gov/programs-surveys/acs" target="acs">(ACS 2019)</a></small>
            </el-col>
            <el-col :span="19">
                <h4>Households with broadband subscription</h4>
                <span class="primary">
                    {{detail.householdsBroadband|formatNumber}}
                    <small>({{(detail.householdsBroadband/detail.households*100) | formatNumber('0.00')}}%)</small>
                </span>
            </el-col>
        </el-row>
        <br>
        <br>

        <el-row :gutter="20" v-if="detail.industries">
            <el-col :span="5">
                <h4>Top Industries</h4>
            </el-col>
            <el-col :span="19">
                <ExportablePlotly :data="industryGraphData" :layout="industryGraphLayout"/>
                <br>
                <br>
            </el-col>
        </el-row>

        <div v-if="!showOtherInfo">
            <el-button round @click="showOtherInfo = true">
                <i class="el-icon-caret-right"/> Show More Information
            </el-button>
        </div>

        <div v-if="showOtherInfo">
            <el-row :gutter="20">
                <el-col :span="5">
                    <h4>Population<br>
                        <small><a href="https://www.census.gov/programs-surveys/popest.html" target="census">(Census Bureau 2019)</a></small>
                    </h4>
                    <span class="primary" v-if="detail.population"> {{detail.population | formatNumber}}</span>
                    <span v-else style="padding: 10px 0; opacity: 0.5;">No information</span>
                </el-col>
                <el-col :span="19">
                    <Histogram v-if="$root.commonReady" :value="detail.population" :histogram="$root.populationHistogram" :fips="fips" :state="detail.state" xunit="people"/>
                    <br>
                    <br>
                </el-col>
            </el-row>

            <el-row :gutter="20">
                <el-col :span="5">
                    <h4>Population Density<br>
                        <small><a href="https://www.census.gov/programs-surveys/acs" target="acs">(ACS 2019)</a></small>
                    </h4>
                    <span class="primary" v-if="detail.popdensity"> {{detail.popdensity | formatNumber}} people per sq. mile</span>
                    <span v-else style="padding: 10px 0; opacity: 0.5;">No information</span>
                </el-col>
                <el-col :span="19">
                    <Histogram v-if="$root.commonReady" :value="detail.popdensity" :histogram="$root.popdensityHistogram" :fips="fips" :state="detail.state" xunit="people per sq. mile"/>
                    <br>
                    <br>
                </el-col>
            </el-row>

            <hr>

            <el-row :gutter="20">
                <el-col :span="5">
                    <h4>GDP<br>
                        <small><a href="https://www.bea.gov/" target="bea">(BEA 2018)</a></small>
                    </h4>
                    <span class="primary" v-if="detail.gdp"> ${{(detail.gdp/1000) | formatNumber}} M</span>
                    <span v-else style="padding: 10px 0; opacity: 0.5;">No information</span>
                    <br>
                </el-col>
                <el-col :span="19">
                    <Histogram v-if="$root.commonReady" :value="detail.gdp" :histogram="$root.gdpHistogram" :fips="fips" :state="detail.state" xunit="dollars"/>
                    <br>
                    <br>
                </el-col>
            </el-row>


            <el-row :gutter="20">
                <el-col :span="5">
                    <h4>Per Capita Income<br>
                        <small><a href="https://www.bea.gov/" target="bea">(BEA 2019)</a></small>
                    </h4>
                    <span class="primary" v-if="detail.percapitaincome"> ${{detail.percapitaincome | formatNumber}}</span>
                    <span v-else style="padding: 10px 0; opacity: 0.5;">No information</span>
                </el-col>
                <el-col :span="19">
                    <Histogram v-if="$root.commonReady" :value="detail.percapitaincome" :histogram="$root.perCapitaIncomeHistogram" :fips="fips" :state="detail.state" xunit="dollars per capita"/>
                    <br>
                    <br>
                </el-col>
            </el-row>

            <el-row :gutter="20">
                <el-col :span="5">
                    <h4>Median Household Income<br>
                        <small><a href="https://www.census.gov/programs-surveys/acs" target="acs">(ACS 2019)</a></small>
                    </h4>
                    <span class="primary" v-if="detail.medianincome"> ${{detail.medianincome | formatNumber}}</span>
                    <span v-else style="padding: 10px 0; opacity: 0.5;">No information</span>
                </el-col>
                <el-col :span="19">
                    <Histogram v-if="$root.commonReady" :value="detail.medianincome" :histogram="$root.medianIncomeHistogram" :fips="fips" :state="detail.state" xunit="dollars"/>
                    <br>
                    <br>
                </el-col>
            </el-row>

            <hr>

            <el-row :gutter="20">
                <el-col :span="5">
                    <h4>Per Capita Money Income<br>
                        <small><a href="https://www.census.gov/programs-surveys/acs" target="acs">(5-year ACS)</a></small>
                    </h4>
                </el-col>
                <el-col :span="19">
                    <p>
                        <ExportablePlotly :data="pcmGraphData" :layout="pcmGraphLayout"/>
                        <small>The amount of money (only cash sources) earned per person. Released annually in December.</small>
                    </p>
                    <br>
                    <br>
                </el-col>
            </el-row>

            <el-row :gutter="20">
                <el-col :span="5">
                    <h4>Per Capita Personal Income<br>
                        <small><a href="https://www.bea.gov/" target="bea">(BEA 2019)</a></small></h4>
                </el-col>
                <el-col :span="19">
                    <p>
                        <ExportablePlotly :data="pcpGraphData" :layout="pcpGraphLayout"/>
                        <small>An estimate of income per person that includes not only cash sources of income, but also insurance, transfer payments, dividends, interest, and rent. Released annually in the spring.</small>
                    </p>
                    <br>
                </el-col>
            </el-row>

        </div>

        <br>

        <hr>

        <div>
            <el-link type="primary" :target="'json.'+fips" :href="$root.dataUrl+'/counties/county.'+fips+'.json'">Download County Detail (.json)</el-link>
        </div>
        <h3>Data Sources</h3>
        <p>
            <ol>
                <li>The population estimate, racial makeup, and population history data are coming from the Census Bureau's Population Estimates by Age, Sex, Race, and Hispanic Origin.</li>
                <li>The Population Density, Median Household Income, and Per Capita Money Income are coming from the Census Bureau's American Community Survey, 5-year estimates.</li>
                <li>The GDP, Per Capita Income, and Per Capita Personal Income (which is the same as the Per Capita Income data that is currently being displayed) are from the Bureau of Economic Analysis.</li>
                <li>The Top Industries data is from the Census of Employment and Wages data from the Bureau of Labor Statistics and the Unemployment Rate data is LAUS from BLS.</li>
            </ol>
        </p>
        <h3>Additional Resources</h3>
        <p>
            <ol>
                <li>Measuring Distress Platform <a href="https://www.statsamerica.org/distress/distress.aspx">(StatsAmerica)</a>.</li>
                <li>USA Opportunity Zones <a href ="https://www.statsamerica.org/opportunity/">(StatsAmerica)</a>.</li>
            </ol>
        </p>
    </div>

    <div class="page" v-if="tab == 'disaster'">
        <br>
        <div class="overview">
            <p>
                This page provides information on FEMA Disaster Declarations for <b>{{detail.county}} County</b> from 1954 to 2020. It also presents information on EDA Disaster Supplemental Awards provided to eligible counties from 2012 to 2020.
                Only FEMA Disaster Declarations are displayed (not Emergency Declarations).        
            </p>
        </div>
        <p v-if="recentHistory.length == 0" style="opacity: 0.8;">No disaster declared since 2017</p>
        <div v-for="(event, idx) in recentHistory" :key="idx" class="history">
            <Event :event="event">
                <div v-if="idx < recentHistory.length" style="padding: 10px 0;">
                    <Eligibility2018 v-if="$root.isEDA && is2018Eligible(event)"/>
                    <Eligibility2019 v-if="$root.isEDA && (is2019Eligible(event) || is2019FloodEligible(event))"/>
                </div>
            </Event>
        </div>

        <div v-if="!showPastHistory" style="border-top: 2px solid #f3f3f3; margin-left: 230px;">
            <br>
            <el-button round @click="showPastHistory = !showPastHistory">
                <i class="el-icon-caret-right"/> Show Old Disaster Declarations ({{pastHistory.length}})
            </el-button>
        </div>

        <!-- don't use slide-down.. it will be slow :active="showPastHistory" :duration="1000"-->
        <div v-if="showPastHistory">
            <Event :event="event" v-for="(event, idx) in pastHistory" :key="idx" class="history"/>
        </div>
        <br>
        <h3>Data Sources</h3>
        <p>
         Data for the disaster declaration was generated from the <a href="https://www.fema.gov/disasters/disaster-declarations"> FEMA website</a> and is updated daily. Data for the EDA award was obtained from
         the EDA and is updated as the data becomes available.
         </p>
        <h3>Additional Resources</h3>
        <p>
            <ol>
                <li>FEMA Hazard Mitigation Plan Status <a href="https://fema.maps.arcgis.com/apps/webappviewer/index.html?id=ec2fb023df744cf480da89539338c386">(FEMA)</a>.</li>
                <li>Resilience Analysis and Planning Tool (RAPT)<a href ="https://fema.maps.arcgis.com/apps/webappviewer/index.html?id=90c0c996a5e242a79345cdbc5f758fc6">(FEMA)</a>.</li>
            </ol>
        </p>
    </div>

    <div v-if="tab == 'bvi'" class="page">
        <br>
        <div class="overview">
            <p>
            The Business Vulnerability Index (BVI) presents information on the percentage of businesses in <b>{{detail.county}} County</b> that is believed to be most 
            vulnerable to various natural disasters. The information comes from the U.S. Census Bureau's County Business Patterns.
            </p>

            <p>
                Businesses that we identified to be especially vulnerable to a disaster have the following characteristics:
                <ol>
                    <li>dependent on supply chains.</li>
                    <li>have a high reliance on public utilities like water and electricity.</li>
                    <li>or have a large infrastructure footprint and low infrastructure mobility.</li>
                </ol>
            </p>
            <p>This information will <b>help practitioners and policymakers</b> in {{detail.county}} County to know the business sectors that <b>deserve more attention</b> in 
            terms of disaster resiliency planning. Some data points are suppressed (missing) due to small employment size to avoid disclosure (confidentiality) or withheld because data do not meet publication standards. 
            </p>
        </div>

        <div class="plot-legend">
            <div class="color-box" style="height: 4px; background-color: #999"/> Not Vulnerable
            <div class="color-box" style="height: 4px; background-color: #900"/> Vulnerable
        </div>
        <br clear="both">

        <br>
        <br>
        <div v-for="(data, naics) in bvi2" :key="naics">
            <p>
                <b><NaicsInfo :id="naics"/></b>
            </p>
            <el-row>
                <el-col :span="12">
                    <h4 style="margin: 0"><small>Establishments</small></h4>
                    <ExportablePlotly :data="data.estPlotly" :layout="bviLayout"/>
                </el-col>
                <el-col :span="12">
                    <h4 style="margin: 0"><small>Employment</small></h4>
                    <ExportablePlotly :data="data.empPlotly" :layout="bviLayout"/>
                </el-col>
            </el-row>
        </div>

        <div v-if="!showNonvBVI && Object.keys(bvi2_nonv).length > 0" style="border-top: 2px solid #f3f3f3;">
            <br>
            <el-button round @click="showNonvBVI = !showNonvBVI">
                <i class="el-icon-caret-right"/> Show Non-vulnerable Sectors ({{Object.keys(bvi2_nonv).length}})
            </el-button>
        </div>

        <!-- don't use slide-down.. it will be slow :active="showPastHistory" :duration="1000"-->
        <div v-if="showNonvBVI" style="border-top: 2px solid #f3f3f3;">
            <br>
            <div v-for="(data, naics) in bvi2_nonv" :key="naics">
                <p>
                    <b><NaicsInfo :id="naics"/></b>
                </p>
                <el-row>
                    <el-col :span="12">
                        <h4 style="margin: 0"><small>Establishments</small></h4>
                        <ExportablePlotly :data="data.estPlotly" :layout="bviLayout"/>
                    </el-col>
                    <el-col :span="12">
                        <h4 style="margin: 0"><small>Employment</small></h4>
                        <ExportablePlotly :data="data.empPlotly" :layout="bviLayout"/>
                    </el-col>
                </el-row>
            </div>
        </div>
        <br clear="both">
    </div>

    <div class="page" v-if="tab == 'resilience'">
        <br>
        <div class="overview">
            <p>
            The Disaster Resilience Index <b>measures the capacity of {{detail.county}} County to recover from disaster events</b> without losing its socioeconomic 
            capacity. This information will help practitioners and policymakers to see where there are strengths and weaknesses within {{detail.county}} county
            in the context of vulnerability to disaster events. These insights are also useful for performing a <b>SWOT Analysis for economic recovery</b>.
            </p>
            <p>
            The resilience scores are calculated using information collected by the <a href="https://data.census.gov/cedsci/">U.S. Census</a> and the framework provided by <a href="http://resiliencesystem.com/sites/default/files/Cutter_jhsem.2010.7.1.1732.pdf">[Cutter et al. 2010]</a>.
            </p>    
        </div>

        <div class="plot-legend">
            <div class="color-box" style="height: 4px; background-color: #409eff"/> This County
            <div class="color-box" style="background-color: #09f5"/> State Average(+standard deviation) 
            <div class="color-box" style="background-color: #0003"/> US Average(+standard deviation) 
        </div>

        <div v-for="(indicator, incode) in detail.cutter2" :key="incode" style="margin-bottom: 15px; clear: both;" class="indicator">
            <div class="indicator-header">
                <b class="indicator-name">{{indicator.name}}</b>
                <p class="indicator-detail">
                    <IndicatorInfo :id="incode"/>
                </p>
            </div>

            <div class="measure-info" v-for="source in detail.cutter2[incode].sources.filter(s=>s.stats)" :key="source.id">
                <p style="margin: 0" :title="source.id">
                    <b>{{source.name}}</b>
                </p>

                <p style="min-height: 50px; margin: 0">
                    {{$root.drMeasures[source.id].desc}}
                </p>
                <CompositePlot v-if="detail" :cutters="source.stats" :edaAwards="detail.eda2018"/>
                <br>

                <p>
                    <el-button size="mini" @click="toggleMeasureDetail(source)" round>
                        <span v-if="source.showDetail">Hide</span>
                        <span v-else>Show</span>
                        Detail
                    </el-button>
                </p>
                <slide-up-down :active="source.showDetail">
                    <small>
                        {{$root.drMeasures[source.id].calcDesc}}
                    </small>
                </slide-up-down>
            </div>
            <br clear="both"> 
        </div>
    </div>

    <div v-if="tab == 'storms'" class="page">
        <br>
        <div v-if="stormData && stormData.length > 0">
            <div class="overview">
                <p>This graph shows the counts of storm events for <b>{{detail.county}} County</b> published by NOAA since the 1950s.</p>
                <p>Storm data has gone through many changes and versions over the years. The source data ingested into the database are widely varied and leads to many questions about the precision and accuracy of the location data. For example, data collected before 1996 was a developing system. Less information was recorded as the database of storm information developed from 1954. Since 1996, a 48-event system was implemented with consistency thus producing more accurate and reliable data. Please see <a href="https://www.ncdc.noaa.gov/stormevents/faq.jsp" target="noaa">https://www.ncdc.noaa.gov/stormevents/faq.jsp</a> for more detail.</p>
                <p>You can click on the chart legend to select or deselect which storm events to show on the graph.</p>
            </div>
            <ExportablePlotly :data="stormData" :layout="stormLayout"/>
        </div>
        <p v-else>No storm data</p>
    </div>

    <div v-if="tab == 'cost'" class="page">
        <div v-if="detail.costDisasters">
            <br>
            <div class="overview">
                <p>
                    Examining GDP from one year before to one year afer a declared disaster for almost 20 years
                    between 2003 and 2019 the following changes in <b>GDP</b> may be expected for {{detail.state}} if a 
                    disaster strikes.
                </p>
            </div>

            <p style="margin-left: 10px;">
                <el-checkbox v-model="showInsigCost">Show industries with statistically insignificant changes</el-checkbox>
            </p>
            <br clear="both">

            <el-row>
                <el-col :span="6" v-for="rec in detail.costDisasters.filter(rec=>showInsigCost||significant(rec.z))" :key="rec.code">
                    <div style="padding-left: 10px; height: 250px;">
                        <div style="padding-bottom: 10px;">
                            <h4 :title="rec.code" style="height: 40px; font-size: 10pt;">{{rec.desc}}</h4>
                            <span class="primary" :style="{color: rec.z>0?'#409EFF':'#F56C6C'}">
                                {{rec.chg_pct|formatNumber("+0.00")}}%
                            </span>
                            <el-tag type="info" v-if="significant(rec.z)" size="mini" :title="'z-score:'+rec.z">Significant! (p&lt;0.3)</el-tag>
                        </div>
                        <p>
                            <small style="opacity: 0.9;">
                                <span v-if="rec.code != '0001'">
                                    This sector accounts for 
                                    <b>{{(detail.gdpPerSector[rec.code]/detail.gdpPerSector['0001'])*100|formatNumber('0.0')}}%</b>
                                    of the county GDP
                                </span>
                                with a potential 
                                <span v-if="rec.z<0">loss</span> 
                                <span v-if="rec.z>=0">gain</span> 
                                in productivity of <b>${{Math.abs(rec.chg_pct)*detail.gdpPerSector[rec.code]*10|formatNumber()}}</b> dollars due to a declared disaster.
                            </small>
                        </p>
                    </div>
                </el-col>
            </el-row>


        </div>
        <br>
        <br>
        <br>
    </div>

    <br>
    <br>
    <br>
    <br>
    <br>

    <Footer/>
</div>
</template>

<script>

import { Component, Vue, Prop, Watch } from 'vue-property-decorator'

import Event from '@/components/Event.vue'
import IndicatorInfo from '@/components/IndicatorInfo.vue'
import Footer from '@/components/Footer.vue'
import CompositePlot from '@/components/CompositePlot.vue'
import BVIPlot from '@/components/BVIPlot.vue'
import NaicsInfo from '@/components/NaicsInfo.vue'
import CountySelecter from '@/components/CountySelecter.vue'

import Eligibility2018 from '@/components/Eligibility2018.vue'
import Eligibility2019 from '@/components/Eligibility2019.vue'
import Histogram from '@/components/Histogram.vue'
import ExportablePlotly from '@/components/ExportablePlotly.vue'

import SlideUpDown from 'vue-slide-up-down'

//import util from '@/util'

Vue.directive('scroll', {
  inserted: function (el, binding) {
    const f = function (evt) {
      if (binding.value(evt, el)) {
        window.removeEventListener('scroll', f)
      }
    }
    window.addEventListener('scroll', f)
  }
})

@Component({
    components: { 
        ExportablePlotly, 
        Event, 
        Eligibility2018, 
        Eligibility2019,
        IndicatorInfo,
        Footer,
        SlideUpDown,
        CompositePlot,
        BVIPlot,
        NaicsInfo,
        CountySelecter,
        Histogram,
    },
})
export default class CountyDetail extends Vue {

    @Prop() fips;

    statemap;
    detail = null;

    tab = "storms";

    history = [];

    /*
    plotlyButtons(p) {
        console.dir(p);
        return [{
            name: 'Save as SVG',
            icon: util.svgIcon,
            click: ()=>{
                util.createSVG(p);
            },
        }]
    }
    */

    bvi2Layout = {
        height: 100,
        margin: {
            l: 50,
            r: 0,
            t: 10,
            b: 30,
            pad: 10,
        },
        //'paper_bgcolor': '#0000',
        //'plot_bgcolor': '#0000',
    }

    bvi2 = {}; //keyed by naics code, then {years, estab, estabV, emp, empV} 
    bvi2_nonv = {}; //bvi2 with all-0 vulnerablility
    showNonvBVI = false;
    bviLayout = null;

    stormLayout = null;
    stormData = null;

    showInsigCost = false;

    showPastHistory = false;
    shownIndicators = [];

    showOtherInfo = false;

    industryGraphLayout = {
        height: 400,
        //width: 900,
        margin: {"t": 30, "b": 30, "l": 400, "r": 0},
        //showlegend: false
        xaxis: {
            title: "Employment (k)",
        },
    }

    get industryGraphData() {
        const non0 = this.detail.industries.filter(r=>r.empl > 0);
        const total = non0.reduce((a,v)=>{ return a+v.empl}, 0);

        //add percentages to each industry name
        non0.forEach(r=>{
            r.name = r.name + " ("+(r.empl/total*100).toFixed(1)+"%)";
        });

        return [
            {
                x: non0.map(r=>r.empl),
                y: non0.map(r=>r.name),
                type: 'bar',
                orientation: 'h',
                hovertemplate: '%{x:,f}',
                name: "",
            },
        ];
    }

    demoGraphData = [];
    demoGraphLayout = {
        height: 200,
        margin: {
            //l: 50,
            r: 280,
            t: 20,
            b: 30,
        },
        //barmode: 'stack',
        //'plot_bgcolor': '#0000',
        //'paper_bgcolor': '#0000',
    }

    orGraphData = [];
    orGraphLayout = {
        height: 200,
        margin: {
            //l: 50,
            r: 280,
            t: 20,
            b: 30,
        },
        //barmode: 'stack',
        //'plot_bgcolor': '#0000',
        //'paper_bgcolor': '#0000',
        //legend: {"orientation": "h"}
    }


    pcmGraphData = [];
    pcmGraphLayout = {
        height: 150,
        margin: {
            l: 40,
            r: 30,
            t: 20,
            b: 30,
        },
        //'plot_bgcolor': '#0000',
        //'paper_bgcolor': '#0000',
        
        //TODO - I should add annotation like this > https://plotly.com/javascript/line-charts/#labelling-lines-with-annotations
        yaxis: {/*title: 'Unemployment Rate',*/ ticksuffix: "$"},
        annotations: [],
    }

    pcpGraphData = [];
    pcpGraphLayout = {
        height: 150,
        margin: {
            l: 40,
            r: 30,
            t: 20,
            b: 30,
        },
        yaxis: {/*title: 'Unemployment Rate',*/ ticksuffix: "$"},
        //'plot_bgcolor': '#0000',
        //'paper_bgcolor': '#0000',
    }

    urGraphLayout = {
        height: 200,
        margin: {
            l: 40,
            r: 30,
            t: 10,
            b: 30,
        },
        //'plot_bgcolor': '#0000',
        //'paper_bgcolor': '#0000',
        legend: {orientation: 'h', side: 'bottom'},
        yaxis: {/*title: 'Unemployment Rate',*/ ticksuffix: "%"},
    }

    uGraphLayout = {
        height: 200,
        margin: {
            l: 40,
            r: 30,
            t: 10,
            b: 30,
        },
        //'plot_bgcolor': '#0000',
        //'paper_bgcolor': '#0000',
        legend: {orientation: 'h', side: 'bottom'},
        //yaxis: {title: 'Employment'},
        /*
        yaxis2: {
            title: 'US Employment',
            //titlefont: {color: 'rgb(148, 103, 189)'},
            //tickfont: {color: 'rgb(148, 103, 189)'},
            overlaying: 'y',
            side: 'right'
        }
        */
    }
    showEmploymentHistory = false;

    cuttersData = {}; //group by incode then {states: {avg, sdev}, us: {avg, sdev}, county} 

    @Watch('fips')
    onFipsChange() {
        //this.$router.replace("/county/"+this.fips);
        this.load();
    }

    toggleIndicator(incode) {
        const pos = this.shownIndicators.indexOf(incode);
        if(~pos) this.shownIndicators.splice(pos, 1);
        else this.shownIndicators.push(incode);
    }  

    toggleMeasureDetail(source) {
        Vue.set(source, "showDetail", !source.showDetail);
    }

    goto(id) {
        const e = document.getElementById(id);
        e.scrollIntoView();
    }

    processHistory() {
        this.history = [];

        if(this.$root.isEDA) {
            this.detail.eda2018.forEach(rec=>{
                rec.grantee = rec.grantee_name+", "+rec.grantee_city+", "+rec.grantee_state;
                rec.date = new Date(rec.grant_award_date);
                rec.type = "eda2018";
                this.history.push(rec);
            });
        }
        
        this.detail.disasters.forEach(rec=>{
            rec.type = "dr";
            rec.date = new Date(rec.incidentBeginDate);
            this.history.push(rec);
        }); 

        this.history.sort((a,b)=>{
            return (b.date - a.date);
        });
    }

    processBVI2() {
        this.bviLayout = {
            height: 150,
            margin: {
                l: 40,
                r: 30,
                t: 10,
                b: 30,
            },
            legend: {
                y: 1.15,
                bgcolor: 'rgba(255, 255, 255, 0)',
                bordercolor: 'rgba(255, 255, 255, 0)',
                orientation: 'h',
            },
            barmode: 'stack',
            xaxis: {
                type: 'category', //show all years
            },
            yaxis: {
                //tickformat: '',
                //range: [0, 100], //no way to set min without enforcing it?
            },
        }

        if(!this.detail.bvis2) return;
        for(const naics in this.detail.bvis2) {
            const data = this.detail.bvis2[naics];
            data.years = data.years.map(year=>parseInt(year));

            data.estabV = data.estab_v;
            data.empV = data.emp_v;

            //see if there is any non-0 vulnerability values
            let vuln = false;
            data.estabV.forEach(v=>{
                if(v != 0) vuln = true;
            });
            data.empV.forEach(v=>{
                if(v != 0) vuln = true;
            });

            const populatedYears = [];
            const populatedEstab = [];
            const populatedEstabV = [];
            const populatedEmp = [];
            const populatedEmpV = [];
            for(let y = 2012; y <= 2019; ++y) {
                populatedYears.push(y);
                const p = data.years.indexOf(y);
                populatedEstab.push(~p?data.estab[p]:0);
                populatedEstabV.push(~p?data.estabV[p]:0);
                populatedEmp.push(~p?data.emp[p]:0);
                populatedEmpV.push(~p?data.empV[p]:0);
            }
            data.years = populatedYears;
            data.estab = populatedEstab;
            data.estabV = populatedEstabV;
            data.emp = populatedEmp;
            data.empV = populatedEmpV;

            const traces = {
                estPlotly: [
                    //non vulnerable
                    {
                        x: data.years,
                        y: data.estab.map((nv, i)=>{
                            const v = nv - data.estabV[i]
                            if(v <= 0) return null;
                            return v;
                        }),
                        name: 'Non Vulnerable',
                        showlegend: false,
                        type: 'bar',
                        marker: {
                            color: '#0004',
                        },
                        hovertemplate: '%{y:,f}',
                    },

                    //vul
                    {
                        x: data.years,
                        y: data.estabV.map(v=>{
                            if(v <= 0) return null;
                            return v;
                        }),
                        name: 'Vulnerable',
                        showlegend: false,
                        type: 'bar',
                        marker: {
                            color: '#6008',
                        },
                        hovertemplate: '%{y:,f}',
                    }
                ],

                empPlotly: [
                    //non vulnerable
                    {
                        x: data.years,
                        y: data.emp.map((nv, i)=>{
                            const v = nv - data.empV[i];
                            if(v <= 0) return null;
                            return v;
                        }),
                        name: 'Non Vulnerable',
                        showlegend: false,
                        type: 'bar',
                        marker: {
                            color: '#0004',
                        },
                        hovertemplate: '%{y:,f}',
                    },

                    //vul
                    {
                        x: data.years,
                        y: data.empV.map(v=>{
                            if(v <= 0) return null;
                            return v;
                        }),
                        name: 'Vulnerable',
                        showlegend: false,
                        type: 'bar',
                        marker: {
                            color: '#6008',
                        },
                        hovertemplate: '%{y:,f}',
                    }
                ]
            }

            if(vuln) this.bvi2[naics] = traces;
            else this.bvi2_nonv[naics] = traces;
        }
    }

    processStorms() {
        this.stormLayout = {
            margin: {
                l: 40,
                r: 30,
                t: 10,
                b: 30,
            },
            legend: {
                y: 1.15,
                bgcolor: 'rgba(255, 255, 255, 0)',
                bordercolor: 'rgba(255, 255, 255, 0)',
                orientation: 'h',
            },
            barmode: 'stack',
        }

        this.stormData = [];
        for(const type in this.detail.storms) {
            const x = [];
            const y = [];
            for(const year in this.detail.storms[type]) {
                x.push(year);
                y.push(this.detail.storms[type][year]);
            }
            this.stormData.push({
                x,
                y,
                name: type,
                type: 'bar',
            });
        }
    }

    processOR() {
        this.orGraphData = [];
        const template = {
            x: this.detail.or.years,
            stackgroup: 'one',
            line: {
                width: 0,
                shape: 'spline',
                smoothing: 0.8,
            },
            mode: 'lines',
            hovertemplate: '%{y:,f}',
        }
        for(const key in this.detail.or.groups) {

            //let's ignore these for now
            if(key == "hispanic") continue;
            if(key == "non_hisp") continue;

            let name = key.toUpperCase();
            if(name == "AIAN") name="American Indian or Alaskan Native";

            this.orGraphData.push(Object.assign({ 
                y: this.detail.or.groups[key],
                name,
                hoverinfo: 'y+text',
            }, template));
        }
    }

    processDemo() {
        this.demoGraphData = [];

        const template = {
            x: this.detail.pops.years,
            stackgroup: 'one',
            line: {
                width: 0,
                shape: 'spline',
                smoothing: 0.8,
            },
            mode: 'lines',
        }

        this.detail.pops.groups.forEach(group=>{
            this.demoGraphData.push(Object.assign({ 
                y: group.y,
                name: group.name,
                //hoverinfo: 'y+text',
                hovertemplate: '%{y:,f} (%{text})',
            }, template));
        });

        //compute percentages
        for(let y = 0;y < this.detail.pops.years.length; ++y) {
            const total = this.demoGraphData.reduce((sum, v)=>sum+v.y[y], 0);
            this.demoGraphData.forEach(v=>{
                if(!v.text) v.text = [];
                if(total == 0) v.text.push(null);
                else v.text.push((v.y[y]/total*100).toFixed(1)+"%");
            });
        }
    }

    processDistress() {
        this.pcmGraphData = [
        {
            x: this.detail.distress_pcm.years,
            y: this.detail.distress_pcm.est,
            'error_y': {
                type: 'data',
                array: this.detail.distress_pcm.moe,
                visible: true,
            },
            name: '',
            hovertemplate: '%{y:,f}',
        },
        ];
        
        this.pcpGraphData = [
        {
            x: this.detail.distress_pcp.years,
            y: this.detail.distress_pcp.data,
            name: '',
            hovertemplate: '%{y:,f}',
        }];
    }

    get urGraphData() {
        return [
            //county 
            {
                x: this.detail.distress_ur.date,
                //y: this.detail.distress_ur.rate.map(v=>v.toFixed(1)),
                y: this.detail.distress_ur.rate,
                //yaxis: 'y2',
                name: 'County Unempl. Rate',
                line: {
                    color: '#409EFF',
                },
            },

            {
                x: this.$root.unempUS.date,
                //y: this.$root.unempUS.rate.map(v=>v.toFixed(1)),
                y: this.$root.unempUS.rate,
                //yaxis: 'y2',
                name: 'US Unempl. Rate',
                line: {
                    color: '#999',
                },
            }
        ];
    }

    get uGraphData() {
        return [
            //county
            {
                x: this.detail.distress_ur.date,
                y: this.detail.distress_ur.employed.map(v=>v/24),
                stackgroup: 'county',
                name: 'Employed',
                fillcolor: 'rgba(0,0,0,0.1)',
                line: {
                    width: 0,
                },
                hovertemplate: '%{y:,f}',
            },
            {
                x: this.detail.distress_ur.date,
                y: this.detail.distress_ur.unemp.map(v=>v/24),
                stackgroup: 'county',
                name: 'Unemployed',
                fillcolor: 'rgba(153, 30, 30, 0.4)',
                line: {
                    width: 0,
                },
                hovertemplate: '%{y:,f}',
            },

            /*
            //US
            {
                x: this.$root.unempUS.date,
                y: this.$root.unempUS.employed,
                stackgroup: 'us',
                yaxis: 'y2',
                name: 'US Employed',
                fillcolor: 'rgba(0, 0, 0, 0.1)',
                line: {
                    width: 0,
                },
                hovertemplate: '%{y:,f}',
            },
            {
                x: this.$root.unempUS.date,
                y: this.$root.unempUS.unemp,
                stackgroup: 'us',
                yaxis: 'y2',
                name: 'US Unemployed',
                fillcolor: 'rgba(153, 30, 30, 0.2)',
                line: {
                    width: 0,
                },
                hovertemplate: '%{y:,f}',
            },
            */
        ];
    }

    get unempRateDate() {
        const l = this.detail.distress_ur.date.length;
        return this.detail.distress_ur.date[l-1];
    }
    get unempRateCounty() {
        const l = this.detail.distress_ur.rate.length;
        return this.detail.distress_ur.rate[l-1];
    }
    get unempRateUS() {
        const l = this.$root.unempUS.rate.length;
        return this.$root.unempUS.rate[l-1];
    }

    mounted() {
        if(this.fips) this.load();
    }

    load() {
        this.showPastHistory = false;
        this.showNonvBVI = false;

        fetch(this.$root.dataUrl+"/counties/county."+this.fips+".json").then(res=>res.json()).then(data=>{
            this.detail = data;

            //remove some categories that are not calculated
            delete this.detail.cutter2["INST"];
            delete this.detail.cutter2["FLOR"];

            this.processHistory();
            this.processBVI2();
            this.processStorms();
            this.processDemo();
            this.processOR();
            this.processDistress();
        });
    }

    handleScroll() {
        const targets = ["disaster", "bvi", "cutter", "storms"];

        //clear active
        targets.forEach(id=>{
            const e = this.$refs["nav-"+id];
            if(e) e.classList.remove("navigator-item-active");
        });

        //set current active
        const e = this.$refs["scrolled-area"];
        const pagemid = e.scrollTop + e.clientHeight/2;
        targets.forEach(id=>{
            const target = document.getElementById(id);
            if(!target) return;
            if(target.offsetTop < pagemid && (target.offsetTop + target.scrollHeight) > pagemid) { 
                this.$refs["nav-"+id].classList.add("navigator-item-active");
            }
        });
    }

    //computed properties
    get recentHistory() {
        return this.history.filter(h=>(h.date >= new Date("2017-01-01")));
    }
    get pastHistory() {
        return this.history.filter(h=>(h.date < new Date("2017-01-01")));
    }

    goback() {
        this.$router.replace("/");
    }

    is2018Eligible(event) {

        // some incidentType is not eligible
        switch(event.incidentType) {
        case "Chemical":
        case "Human Cause":
        case "Terrorist":
        case "Toxic Substances":
        case "Biological": //TODO confirm with Jason
            return false;
        }

        const eligBegin = new Date("2017-01-01");
        const eligEnd = new Date("2018-01-01");

        // check data range
        let begin = new Date();
        if(event.incidentBeginDate) begin = new Date(event.incidentBeginDate);
        let end = new Date();
        if(event.incidentEndDate) end = new Date(event.incidentEndDate);
        if( !(begin >= eligBegin && begin < eligEnd) &&
            !(end >= eligBegin && end < eligEnd) ) return false;

        return true;
    }

    is2019Eligible(event) {
        // ignore some incidentType 
        switch(event.incidentType) {
        case "Chemical":
        case "Human Cause":
        case "Terrorist":
        case "Toxic Substances":
        case "Biological": //TODO confirm with Jason
            return false;
        }

        const eligBegin = new Date("2018-01-01");
        const eligEnd = new Date("2019-01-01");

        //check date range
        let begin = new Date();
        if(event.incidentBeginDate) begin = new Date(event.incidentBeginDate);
        let end = new Date();
        if(event.incidentEndDate) end = new Date(event.incidentEndDate);
        if( !(begin >= eligBegin && begin < eligEnd) &&
            !(end >= eligBegin && end < eligEnd) ) return false;


        return true;
    }

    is2019FloodEligible(event) {
        switch(event.incidentType) {
        case "Tornado":
        case "Flood":
        case "Severe Storm(s)":
            //good
            break;
        default:
            // also qualifies if incidentTitle includes those names
            if( 
                event.incidentTitle &&
                !event.incidentTitle.toLowerCase().includes("tornado") &&
                !event.incidentTitle.toLowerCase().includes("flood") &&
                !event.incidentTitle.toLowerCase().includes("severe storms")) return false;
        }

        const eligBegin = new Date("2019-01-01");
        const eligEnd = new Date("2020-01-01");

        //check date range
        let begin = new Date();
        if(event.incidentBeginDate) begin = new Date(event.incidentBeginDate);
        let end = new Date();
        if(event.incidentEndDate) end = new Date(event.incidentEndDate);
        if( !(begin >= eligBegin && begin < eligEnd) &&
            !(end >= eligBegin && end < eligEnd) ) return false;

        return true;
    }

    contextMenuCounty = null;
    openContextMenuCounty() {
        window.open("#/county/"+this.contextMenuCounty, "apred-"+this.contextMenuCounty);
    }

    scroll2top() {
        window.scrollTo(0, 0);
    }

    compare() {
        window.open("#/compare/what/"+this.fips, "compare-what-"+this.fips);
    }

    significant(z) {
        //return (z < -1.64485 || z > 1.64485); //p<0.1
        //return (z < -1.28155 || z > 1.28155); //p<0.2
        return (z < -1.03643 || z > 1.03643); //p<0.3
    }
}

</script>

<style lang="scss" scoped> 
.indicator + .indicator {
    border-top: 1px solid #0002;
}
.indicator-header {
    position: relative; 

    margin-top: 20px;
    padding-top: 20px;
    margin-bottom: 20px;

    .indicator-name {
        font-size: 130%;
    }
    .indicator-detail {
        opacity: 0.9;
    }
}
p {
    margin-top: 0px;
    line-height: 170%;
    color: #666;
}
h2 {
    padding: 20px 0;
    color: #0006;
    text-transform: uppercase;
}

h3 {
    padding: 20px 0 0 0;
    color: #0006;
    text-transform: uppercase;
    font-size: 23px;
    font-weight: bold;
}

h4 {
    opacity: 0.7;
    margin-bottom: 10px;
}
h5 {
    margin-bottom: 5px;
}

.sub-heading {
    opacity: 0.6;
    font-weight: bold;
    font-size: 90%;
    color: black;
    white-space: nowrap;
}
.primary {
    font-weight: bold;
    font-size: 150%;
}

.important, 
.primary {
    color: #409EFF;
}
.border-left {
    border-left: 1px solid #0001;
    height: 200px;
    padding-left: 20px;
    padding-top: 10px;
}
@media only screen and (max-width: 650px) {
    .demo {
        display: none;
    }
}
@media only screen and (max-width: 1000px) {
    .gdp {
        display: none;
    }
}
</style>

<style lang="scss"> 
.el-collapse,
.el-collapse-item__header {
    border: none;
}
.header {
    position: sticky;
    top: 50px;
    padding-top: 10px;
    width: 100%;
    background-color: white;
    z-index: 3;
}

/*
#info-header {
    background-color: #eee;
    height: 220px;
}
*/

.header h3 {
    margin: 0;
    padding-top: 10px;
}
.header .back-button {
    position: relative;
    font-size: 100%;
    margin-right: 20px;
}
/*
.sidebar {
    position: sticky; 
    top: 10px;
}
*/
.navigator {
    margin-top: 20px;
    .navigator-item {
        color: gray;
        cursor: pointer;
        padding: 10px 10px;
        margin: 0;
        border-left: 3px solid #0000;
    }
    .navigator-item:hover {
        background-color: #eee;
    }
    .navigator-item-active {
        color: #409EFF;
        border-color: #409EFF;
    }
}
/*
@media only screen and (max-width: 1500px) {
    .sidebar {
        display: none;
    }
}
#statemap {
    position: relative;
    width: 100%;
    height: 220px;
}
*/
canvas:focus {
    outline: none;
}
.indicator-detail {
    clear: both;
}
small {
    opacity: 0.5;
}
.measure-info {
display: inline-block;
float: left;
width: 475px;
margin-right: 20px;
margin-bottom: 10px;
}
.bvi {
display: inline-block;
float: left;
width: 475px;
margin-right: 20px;
margin-bottom: 10px;
}
.measure-info:nth-child(even) {
clear: both;
}
.plot-legend {
    text-align: center;
    font-size: 85%;
    .color-box {
        margin-left: 20px;
        display: inline-block; 
        width: 16px; 
        height: 16px; 
    }
}
h4 {
    margin-top: 0;    
}
hr {
    opacity: 0.3;
    margin-bottom: 30px;
}
.overview {
    background-color: #eee;
    padding: 20px;
    margin-bottom: 20px;
    border-radius: 10px;
    font-size: 95%;
    p:last-child {
        margin-bottom: 0;
    }
}
.toolset {
    float: right; 
    width: 375px; 
    padding-top: 15px;
}
</style>
