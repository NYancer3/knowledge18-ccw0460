# Lab 1: Building the Base Widget
In our first lab, we will look at using Dependencies to build widgets that use third-party libraries for additional functionality. We will use this initial widget in our final product, while building in more features as we go.
## Lab Goal
Understand **Widget Dependencies** and how they can be used to provide functionality to your widgets. Learn how to use **Widget Options** to configure the presentation and behavior of your widget, making it reusable.
## Create a Widget Dependency
1. Navigate to **Service Portal** > **Dependencies**
2. Click the **New** button at the top of the list
3. Fill in the following fields:
    1. Name: **Angular Chart JS 2.X**
    1. Angular module name: **angular-chart-js-2-x**
4. Click **Submit**
## Now, Add JS Includes
1. Open the Dependency you just created
1. Under JS Includes at the bottom of the form > Select **New**
1. Fill in the form as follows:
    1. Display Name: **ChartJS**
    1. Source: **URL**
    1. URL: **https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.7.2/Chart.bundle.min.js**
1. Click **Submit**
1. Under JS Includes at the bottom of the form > Select **New**
1. Fill in the form as follows:
    1. Display Name: **Angular ChartJS**
    1. Source: **URL**
    1. JS File URL: **https://cdn.jsdelivr.net/angular.chartjs/1.1.1**
1. Click **Submit**
1. Change the Order of the “cdn.jsdeliver.net” Include to **200**
## Now Let's Create a Widget to Use It
1. Navigate to **Service Portal** > **Widgets**
1. Click **New**
1. Fill in the form as follows:
    1. Name: **Donut Chart 2.0**
    2. ID: **donut-chart-20**
    3. Body HTML Template:
    ```html
    <div class="chartContainer">
      <h3 class="text-center" ng-if="!!c.data.options.chartTitle">{{ c.data.options.chartTitle }}</h3>
      <canvas ng-if="!!c.data.chartData" id="pie" class="chart chart-doughnut" chart-data="c.data.chartData" chart-options="c.data.chartOptions" chart-labels="c.data.chartLabels" chart-click="c.onClick" chart-colors="c.data.chartColors">
      </canvas>
      <p ng-if="c.data.chartData == ''">No data to display</p>
    </div>
    ```
    4. CSS:
    ```css
    div.chartContainer {
      max-width: 500px;
      padding: 8px;
      margin: auto !important; 
    }
    ```
    5. Server Script:
    ```javascript
    (function() {
      /* populate the 'data' object */
      /* e.g., data.table = $sp.getValue('table'); */

      // Instantiate a container for our chart options. We will get these from the widget instance options.
      data.options = {};

      // Here we define the table, "group by" field, and initial filter for the chart
      var table = $sp.getValue('table') || options.table;
      var field = $sp.getValue('display_field') || options.display_field;
      var filter = $sp.getValue('filter') || options.filter;

      data.currentPage = $sp.getParameter('id');
      
      // Instantiate axis labels and data containers.
      data.chartLabels = [];
      data.chartData = [];
      
      // We will need to make an index of field values to query conditions. We will need this for the click actions on our chart; when you click the chart, we will get the query value to send to the list based on the value of the slice you clicked.
      data.chartLookup = {};
      
      // We can either specify a target page for our chart, or default to the "list" page
      data.targetPage = options.target_page || 'list';

      // Here we start setting our chart options. Note the defaults in case you do not enter any instance options
      // Animation Speed, default 1000
      data.options.animate = parseInt(options.animation_speed);
      if (isNaN(parseInt(data.options.animate))){
        data.options.animate = 1000;
      }

      // donut_coutout_percent, default 50, min 0, max 90
      data.options.donut_cutout = options.donut_cutout_percent;
      data.options.orig_cutout = data.options.donut_cutout;
      if (isNaN(parseInt(data.options.donut_cutout)) || data.options.donut_cutout < 0 || data.options.donut_cutout > 90){
        data.options.donut_cutout = 50;
      }

      // Arc percent, default 0
      data.options.chartRotation = options.rotation_offset || 0;

      // Chart circumference, default 100. Min 25, max 100.
      data.options.circ = options.arc_percent;
      data.options.chartCircumference = parseInt(options.arc_percent) || 100;
      if (isNaN(parseInt(data.options.chartCircumference)) || data.options.chartCircumference < 25 || data.options.chartCircumference > 100){
        data.options.chartCircumference = 100;
      }

      // We can use the built-in chart colors settings. This allows you to define colors per table to be used in this or other out-of-the-box rerports
      function getColors(tbl,fld){
        var retArr = [];
        var g_colors = new GlideChartFieldColors(tbl,fld);
        data.chartLabels.forEach(function(lbl){
          var val = data.chartLookup[lbl];
          var col = val.split('=');
          col = (col.length > 1 ? col[1] : col[0]).toString();
          retArr.push(g_colors.get(col));
        });

        console.log(retArr);

        return retArr;
      }

      // If we have defined a table and field for this widget in the instance options, get the data to build the chart
      if (!!table && !!field){


        data.options.chartTitle = $sp.getValue('title') || options.title;

        data.options.table = table;
        data.options.display_field = field;
        data.options.filter = filter;

        var ga = new GlideAggregate(table);
        ga.addAggregate('COUNT',field);
        ga.orderByAggregate('COUNT',field);
        ga.addEncodedQuery(filter);
        ga.query();

        var oth = 0;
        var othItems = [];

        // We are setting a hard limit of 12 items for an "other" group. How might you make this configurable?
        while (ga.next() && oth < 12){
          var disp = !!ga.getDisplayValue(field) ? ga.getDisplayValue(field) : '(Empty)';
          data.chartLabels.push(disp);
          data.chartLookup[disp] = '=' + ga.getValue(field);
          othItems.push(ga.getValue(field));
          data.chartData.push(ga.getAggregate('COUNT',field));
          oth++;
        }

        // If we have reached our threshold for "Other," then our query should be anything that does not match what we have used so far
        if (oth == 12){
          var gr = new GlideRecord(table);
          gr.addEncodedQuery(filter);
          gr.addQuery(field,'NOT IN',othItems.toString());
          gr.query();

          if (gr.getRowCount() > 0){
            data.chartLabels.push('Other');
            data.chartLookup['Other'] = 'NOT IN' + othItems;
            data.chartData.push(gr.getRowCount());
          }
        }

        data.chartColors = getColors(table,field);

      }

    })();
    ```
    5. Client controller:
    ```javascript
    function(spUtil,$scope,$rootScope,$location) {
      /* widget controller */
      var c = this;

      // Here we define how the chart legend is displayed. How might you make this configurable?
      c.data.chartLegend = {
        display:true,
        position:'bottom',
        labels:{
          boxWidth:15
        }
      };

      // Here we are building our chart options. This controls the visual display and animations for the chart.
      c.data.chartOptions = {
        rotation:((-0.5 + (-c.data.options.chartRotation)/50)*Math.PI),
        circumference:2*(c.data.options.chartCircumference/100)*Math.PI,
        legend:c.data.chartLegend,
        animation:{
          duration: c.data.options.animate,
          easing:'easeOutBounce'
        },
        cutoutPercentage:c.data.options.donut_cutout
      };

      // If we have a lot of text in our legend labels, decrease the font size for better display.
      if (c.data.chartLabels.toString().length > 250){
        c.data.chartLegend.labels.fontSize = 10;
      }

      // Here we define what happens when you click on the chart. The points argument contains information about the area that was clicked, and the evt argument contains information about the event itself. 
      c.onClick = function (points, evt) {

        // Check to see if we have anything in the points array
        if (points.length > 0){
          // The first member of the points array contains information about the portion of the chart we clicked. The _model property does not contain a "value," just a label which is why we built the "chartLookup" object to index based on display value.  
          var filt = (!!c.data.options.filter ? c.data.options.filter + '^' : '') + c.data.options.display_field + c.data.chartLookup[points[0]._model.label];
          
          // Now we build an object for the click target. This will be used to add parameters to the URL location
          var targObj = {
            id: c.data.targetPage,
            table:c.data.options.table,
            filter:filt,
            previous_id:c.data.currentPage,
            view:'sp'
          };
          $location.search(targObj);
        }

      };

      // We want to watch for any changes, so we will use the recordWatch functionality to keep the chart up to date
      if (!!c.data.options.table){
        spUtil.recordWatch($scope,c.data.options.table,c.data.options.filter);
      }

    }
    ```
## Let's Make the Widget Reusable by Adding Options
1. Open the **Donut Chart 2.0** widget record (if you were not directed there from the previous step)
1. Scroll to the “Related Links” section of the form and click **Open in Widget Editor**
1. Click the Menu icon in the upper-right-hand corner (three horizontal lines) and then select **Edit option schema**
1. Add the following option (click the “plus” icon at the top of the window to add options):
    1. Label: **Donut Cutout Percent**
    1. Name (field name syntax): **_[Auto populates]_**
    1. Type: **Integer**
    1. Hint: **Size of the “donut hole,” default is 50%**
    1. Form section: **Presentation**
1. Add a second option (click the “plus” icon at the top of the window to add options):
    1. Label: **Arc Percent**
    1. Name (field name syntax): **_[Auto populates]_**
    1. Type: **Integer**
    1. Hint: **Percent of a circle the chart should consume. Min 25%**
    1. Form section: **Presentation**
1. Add a third option (click the “plus” icon at the top of the window to add options):
    1. Label: **Rotation Offset**
    1. Name (field name syntax): **_[Auto populates]_**
    1. Type: **Integer**
    1. Hint: **Percentage to rotate the chart from 0 degrees vertical**
    1. Form section: **Presentation**
1. Add a fourth option (click the “plus” icon at the top of the window to add options):
    1. Label: **Animation Speed**
    1. Name (field name syntax): **_[Auto populates]_**
    1. Type: **Choice**
    1. Choices (Note that each choice needs to be on a separate line):
          
          **0**
          
          **500**
          
          **750**
          
          **1000**
          
          **1250**
          
          **1500**
    1. Hint: **Number of milliseconds it takes to animate the chart**
    1. Form section: **Presentation**
1. Add a fifth option (click the “plus” icon at the top of the window to add options):
    1. Label: **Target Page**
    1. Name (field name syntax): **_[Auto populates]_**
    1. Type: **String**
    1. Hint: **Identify the page to target when the chart is clicked. Default is “list”**
    1. Form section: **Behavior**
1. Select **Save**
## Finally, Let's Create a Page for Our Widget to Live
1. Close the **Widget Editor** tab
1. Navigate to **Service Portal** > **Pages** 
1. Select **New**
1. Fill in the form as follows:
    1. Title: **CMDB Dashboard**
    1. ID: **k18_cmdb_dashboard**
1. Select **Submit**
1. Open the new **CMDB Dashboard** page
1. Under the Related Links section, select **Open in Designer** (New tab will open)
1. Drag a **“12” Column** to the existing container in the designer
1. In the filter type: **donut**
1. Drag and drop the **Donut Chart 2.0** widget into the Column
1. Click the **Edit** icon (![Edit Widget Icon](/images/edit_widget_v1.jpg)) in the upper right hand corner of the widget
1. In the Modal window that appears fill in the following:
    1. Table: **Windows Server [cmdb_ci_win_server]**
    1. Display field: **Operating System**
    1. Title: **Windows Servers**
    1. Donut Cutout Percent: **25**
    1. Arc Percent: **100**
    1. Rotation Offset: **0**
    1. Animation Speed: **75**
1. Select **Save**
## See What You Just Built!
In the upper right hand corner select the *Open in New Window* icon (![Open Page in New Tab](/images/open_page_v1.jpg)) to open your page in a new window
# Lab 2: Combining and Embedding Widgets
In this lab section, we will look at how to use embedded widgets to build a dynamic user experience. We will make these widgets "talk" to each other in the next lab.
## Lab Goal
Learn how to embed widgets and dynamically show or hide them by watching a common, scope-inherited property.
## Create a Widget Family Tree
1. Navigate to **Service Portal** > **Widgets**
1. Find and open the **Data Table from Instance Definition** widget
1. Select **Clone Widget** in the upper right-hand corner
1. In the Widget list, find and open the **Copy of Data Table from Instance Definition** widget
1. Update the form with the following:
    1. Name: **Combined Donut and Table**
    2. Description: **Embeds the data table and donut chart in a single widget**
    3. Body HTML template:
    ```html
    <div>
        <div class="alert alert-danger" ng-if="data.invalid_table">
            ${Table not defined} '{{data.table_label}}'
        </div>
        <!-- Embed our chart widget, if we were able to generate one -->
        <sp-widget ng-if="!!data.donutWidget" widget="data.donutWidget"></sp-widget>
        <!-- Embed our data table widget, if we were able to generate one -->
        <sp-widget ng-if="data.dataTableWidget" widget="data.dataTableWidget"></sp-widget>
    </div>
    ```
    4. Server Script:
    ```javascript
    (function(){
        /*  "use strict"; - linter issues */
        
        /* Most of the following server script comes from the out-of-the-box data table widget */
        
        // populate the 'data' object
        var sp_page = $sp.getValue('sp_page');
        var pageGR = new GlideRecord('sp_page');
        pageGR.get(sp_page);
        data.page_id = pageGR.id.getDisplayValue();
        $sp.getValues(data);
        if (data.field_list) {
            data.fields_array = data.field_list.split(',');
        } else {
            data.field_list = $sp.getListColumns(data.table);
        }

        data.display_field = $sp.getValue('display_field');

        if (input) {
            data.p = input.p;
            data.o = input.o;
            data.d = input.d;
            data.q = input.q;
        }
        data.p = data.p || 1;
        data.o = data.o || $sp.getValue('order_by');
        data.d = data.d || $sp.getValue('order_direction');

        data.page_index = (data.p - 1);
        data.window_size = $sp.getValue('maximum_entries') || 10;
        data.window_start  = (data.page_index * data.window_size);
        data.window_end = (((data.page_index + 1) * data.window_size));
        data.filter = $sp.getValue("filter");

        var gr = new GlideRecordSecure(data.table);
        if (!gr.isValid()) {
            data.invalid_table = true;
            data.table_label = data.table;
            return;
        }
        data.table_label = gr.getLabel();

        // This is where our code begins.
        // First we will build an object to feed in the options for our data table widget.
        // The options we define for this widget instance are a combination of the options needed for the data table and donut chart
        var widgetParams = {
            table: data.table,
            fields: data.field_list,
            o: data.o,
            d: data.d,
            filter: data.filter,
            window_size: data.window_size,
            view: 'sp',
            title: options.title,
            show_breadcrumbs: true
        };
        
        // Now we will use the server-side getWidget method to get a data table widget we can embed
        data.dataTableWidget = $sp.getWidget('widget-data-table', widgetParams);

        // We will do the same for the donut chart widget. Build an object to feed in the options
        var donutParms = {
            table: data.table,
            filter: data.filter,
            display_field: data.display_field
        };

        // Copy options from the current widget instance to feed into our embedded widget
        for (var opt in options){
            donutParms[opt] = options[opt];
        }

        // Now get the donut chart widget to embed
        data.donutWidget = $sp.getWidget('donut-chart-20',donutParms);
    })();
    ```
    5. Option schema (this is a copy of the options we added to the Donut Chart widget):
    ```javascript
    [{"hint":"Size of the \"donut hole,\" default is 50%","name":"donut_cutout_percent","section":"Presentation","label":"Donut Cutout Percent","type":"integer"},{"hint":"Percent of a circle the chart should consume. Min 25%","name":"arc_percent","section":"Presentation","label":"Arc Percent","type":"integer"},{"hint":"Percentage to rotate the chart from 0 degrees vertical","name":"rotation_offset","section":"Presentation","label":"Rotation Offset","type":"integer"},{"hint":"Number of milliseconds it takes to animate the chart","name":"animation_speed","section":"Presentation","label":"Animation Speed","type":"choice","choices":[{"label":"0","value":"0"},{"label":"500","value":"500"},{"label":"750","value":"750"},{"label":"1000","value":"1000"},{"label":"1250","value":"1250"},{"label":"1500","value":"1500"}]},{"hint":"Identify the page to target when the chart is clicked. Default is \"list\"","name":"target_page","section":"Behavior","label":"Target Page","type":"string"}]
    ```
    6. Fields: **Table, Filter, Fields, Maximum Entries, Order by, Order direction, Title, Bootstrap color, Glyph, Link to this page, Display field**
1. Select **Update**
## Update the CMDB Dashboard Page with Your New Widget
1. Navigate to **Service Portal** > **Pages**
1. Open the **CMDB Dashboard** page
1. Under the Related Links section, select **Open in Designer** (New tab will open)
1. Remove the **Donut Chart 2.0** widget by selecting the trash can in the upper right-hand corner of the widget
1. In the filter, start typing: **Combined Donut and Table**
1. Drag and drop the **Combined Donut and Table** widget into the Column
1. Click the **Edit** icon (![Edit Widget Icon](/images/edit_widget_v1.jpg)) in the upper right-hand corner of the widget
1. In the window that appears, fill in the following:
    1. Table: **Windows Server [cmdb_ci_win_server]**
    1. Fields: **Name, Operating System, Short Description, Operational Status**
    1. Maximum entries: **10**
    1. Display field: **Operating System**
    1. Order by: **Name**
    1. Title: **Windows Servers**
    1. Donut Cutout Percent: **25**
    1. Arc Percent: **50**
    1. Rotation Offset: **25**
    1. Animation Speed:  **750**
1. Select **Save**
## Try it out!
1. In the upper right-hand corner select the *Open in New Window* icon (![Open Page in New Tab](/images/open_page_v1.jpg)) to open your page in a new window
2. You should now see something like the following:
![CMDB Dashboard Page](/images/cmdb_dashboard_1_v1.png)
## Let's Make it Dynamic!
Okay, we've now got two widgets on the page. Wouldn't it be nice if we could show just the chart **OR** the table? Let's make a toggle to choose which widget to display. This will make use of another **Widget Dependency** but will also show you how to code a Directive. You can also make a Directive as an **Angular Provider,** but because this directive relies on specific CSS we are going to package it all up as a Dependency.
1. Navigate to **Service Portal** > **Dependencies**
1. Select **New**
1. Fill out the form as follows:
    1. Name: **css-toggle-switch**
    2. Angular module name: **css-toggle-switch**
1. Select **Submit**
1. Open the **css-toggle-switch** dependency you just created
1. Under the CSS Includes related list, select **New**
1. Fill out the form as follows:
    1. Name: **css-toggle-switch.css**
    1. Source: **URL**
    1. CSS file URL: **https://cdn.jsdelivr.net/npm/css-toggle-switch@4.1.0/dist/toggle-switch.css**
1. Select **Submit**
1. Under JS Includes related list, select **New**
1. Fill out the form as follows:
    1. Display name: **Toggle Switch JS**
    2. Source: **UI Script**
    3. UI Script: [Click the Magnifying Glass > Select New]
        1. API Name: **toggleSwitch**
        2. Script:
        ```javascript
        angular.module('css-toggle-switch',[]).directive('toggleSwitch',[function(){
            // We can define default values in order to make certian directive attributes optional
            var defaults = {
                labelTrue: 'On',
                labelFalse: 'Off'
            };

            return {
                restrict: 'E', // This directive must be called as an HTML element
                scope: {
                    watchVar: '=', // The watch-var attrribute of the element should pass in a variable, expecting an object
                    title: '@' // the title attribute of the element should resolve to a string
                },
                template: '<div>' +
                '<label class="switch-light" onclick="">' +
                '<input type="checkbox" ng-model="watchVar">' +
                '<strong>' +
                '{{ title }}' +
                    '</strong>' +
                    '<span class="alert alert-light">' +
                        '<span>{{ labelFalse }}</span>' +
                        '<span>{{ labelTrue }}</span>' +
                        '<a class="btn btn-primary"></a>' +
                    '</span>' +
                '</label>' +
                '</div>' +
                '<style>' +
                '.switch-light {' +
                    ' max-width: 125px; '+
                    '}' +
                    '.switch-light .alert-light {' +
                    '      border: 1px solid #e2e2e2;' +
                    '}' +
                '</style>',
                compile: function(){
                    // We use the compile method to handle assigning the default values we defined at the start of this directive, if necessary
                    return {
                    pre: function(scope,el,attrs){
                    // Set defaults for labels if nothing passed into the directive
                    scope.labelTrue = attrs.labelTrue || defaults.labelTrue;
                    scope.labelFalse = attrs.labelFalse || defaults.labelFalse;
                    },
                    post: function(scope,el,attrs){

                    }
                };
            }
            };
        }]);
        ```
1. Select **Submit** (in the pop-up)
1. Select **Submit** (in the JS Include Form)  
>Note that if you wanted to make the above Directive an Angular Provider, you would just need to put the *function* portion of the code into the script field and use the same value (toggleSwitch) in the **Name** field of the provider record. However, as we mentioned previously this toggle switch relies on a css library. We could embed that in the "template" property of the directive, but that would get messy. If we were to build this as and Angular Provider, then we would also have to remember to include the CSS dependency for any widget that used the provider. It's just simpler to package both the Directive (JavaScript) and the CSS into a single Widget Dependency.   
## Add the Toggle Switch to the Widget
1. Navigate to **Service Portal** > **Widgets**
1. Find and open the **Combined Donut and Table** widget you created previously
1. Under the Dependencies related list, select **Edit…**
1. Locate the **css-toggle-switch** dependency in the “Collection” container (left)
1. Add it to the “Dependencies List” container (right)
1. Select **Save**
1. Update the Widget Form as follows:
    1. Body HTML Template:
    ```html
    <div>
      <div class="alert alert-danger" ng-if="data.invalid_table">
        ${Table not defined} '{{data.table_label}}'
      </div>
      <!-- Toggle between chart and table view. The toggle will set the value of the showTable variable to true or false when clicked. Notice that we are binding the watchVar scope property in our directive to the current scope's showTable property through the watch-var attribute. -->
      <toggle-switch watch-var="showTable" title="Chart or Table" label-true="Table" label-false="Chart"></toggle-switch>
      <!-- If showTable is false, show the chart. Note the use of ng-if here. The angular chartJS code checks on the chart at regular intervals.
            If the chart canvas is hidden when that check is performed there is an error and the next time you try to show the chart it will not render correctly. -->
      <div ng-if="!showTable">
        <sp-widget ng-if="!!data.donutWidget" widget="data.donutWidget"></sp-widget>
      </div>

      <!-- If showTable is true, then show the data table widget. Note that it is okay to use ng-show here,
        as there is no requirement that the data table is visible for other processes. -->
      <div ng-show="showTable">
        <sp-widget ng-if="data.dataTableWidget" widget="data.dataTableWidget"></sp-widget>
      </div>
    </div>
    ```
    2. Client controller:
    ```javascript
    function ($scope, spUtil, $location, spAriaFocusManager) {
        /* This is the original code copied from the data table widget */
        $scope.$on('data_table.click', function(e, parms){
            var p = $scope.data.page_id || 'form';
            var s = {id: p, table: parms.table, sys_id: parms.sys_id, view: 'sp'};
            var newURL = $location.search(s);
            spAriaFocusManager.navigateToLink(newURL.url());
        });

        /* Here is our code. We are just defining the "showTable" property for $scope. Defaulting to false. */
        $scope.showTable = false;
    }
    ```
1. Select **Update**
## Try it out!
1. Navigate back to the **CMDB Dashboard** page (the url should be https://[your-instance-url]/sp?id=k18_cmdb_dashboard). If you already have it open in another tab or window, refresh the page.
1. You should now see something like the following:
![CMDB Dashboard Chart Only](/images/cmdb_dashboard_2_v1.png)
1. Click the “Chart or Table” toggle to change it to **Table**. Your page should now look something like the following:
![CMDB Dashboard Table Only](/images/cmdb_dashboard_3_v2.png)
# Lab 3: Inter-Widget Communication
Now that we have our widgets on the same page, wouldn't it be nice if the table showed the records related to the portion of the chart that we click instead of navigating when we click the chart? You bet it would! So let's make it do that.
## Lab Goal
Learn how to use events to pass data between different widgets on a page.
## Teach Your Widgets to be Active Listeners
1. Navigate to **Service Portal** > **Widgets**
1. Open the **Donut Chart 2.0** widget you created previously
1. Modify the form as follows:
    1. Server Script:
    ```javascript
	(function() {
		/* populate the 'data' object */
		/* e.g., data.table = $sp.getValue('table'); */

		// Instantiate a container for our chart options. We will get these from the widget instance options.
		data.options = {};

		// Here we define the table, "group by" field, and initial filter for the chart
		/*
			LAB 3: Add "input" properties to our assignments. This will allow the server script to be called asynchronously from the client script.
		*/
		var table = $sp.getValue('table') || options.table || input.table;
		var field = $sp.getValue('display_field') || options.display_field || input.display_field;
		var filter = $sp.getValue('filter') || options.filter || input.filter;
		/* END LAB 3 CHANGE */
		
		/*
			LAB 3: We need to set some properties on the "data" object.
			These will passed back to this script as "input" from the client.
			Necessary for asynchronous functionality.
		*/
		data.table = table;
		data.filter = filter;
		data.display_field = field;
		/* END LAB 3 CHANGE */
		
		/*
			LAB 3: Are we event-driven? If nothing given, default to false.
		*/
		data.options.event_driven = options.event_driven || input.event_driven || 'false';
		data.event_driven = data.options.event_driven;
		/* END LAB 3 CHANGE */
		
		data.currentPage = $sp.getParameter('id');

		// Instantiate axis labels and data containers.												 
		data.chartLabels = [];
		data.chartData = [];

		// We will need to make an index of field values to query conditions. We will need this for the click actions on our chart; when you click the chart, we will get the query value to send to the list based on the value of the slice you clicked.
		data.chartLookup = {};

		// We can either specify a target page for our chart, or default to the "list" page
		/*
			LAB 3: Note the addition, again, of "input" to our checks for populating this property.
		*/
		data.targetPage = options.target_page || input.target_page || 'list';
		/* END LAB 3 CHANGE */

		// Here we start setting our chart options. Note the defaults in case you do not enter any instance options
		// Animation Speed, default 1000
		/*
			LAB 3: We are going to put animation speed into a variable.
			In the original script, we werer getting this from "options."
			However, we need to also check "input" for a value, so we do that in a variable.
			Note that we are using the variable on the very next line.
		*/
		var anim_speed = options.animation_speed || input.animation_speed;
		data.options.animate = parseInt(anim_speed);
		/* END LAB 3 CHANGE */
        
		if (isNaN(parseInt(data.options.animate))){
			data.options.animate = 1000;
		}

		/*
			LAB 3: We are having some issues when the value is 0.
			Also, note that we are also checking "input" for the donut_coutout_percent value.
		*/
		if (!!options && options.donut_cutout_percent == 0){
			data.donut_cutout_percent = '0';
		} else if (!!input && input.donut_cutout_percent == 0){
			data.donut_cutout_percent = '0';
		} else {
		data.donut_cutout_percent = options.donut_cutout_percent || input.donut_cutout_percent;
		}
		/* END LAB 3 CHANGE */
		
		// donut_coutout_percent, default 50, min 0, max 90
		/*
			LAB 3: Note that we are setting the property from "data" this time (set above)
			and not from "options" directly.
			This is because a value of zero was previously causing issues AND because we want to include "input" in our checks for this value.
		*/
		data.options.donut_cutout = data.donut_cutout_percent;
		/* END LAB 3 CHANGE */
		
		data.options.orig_cutout = data.options.donut_cutout;
		if (isNaN(parseInt(data.options.donut_cutout)) || data.options.donut_cutout < 0 || data.options.donut_cutout > 90){
			data.options.donut_cutout = 50;
		}

		// Arc percent, default 0
		/*
			LAB 3: Note that we are also checking "input" for this value.
		*/
		data.options.chartRotation = options.rotation_offset || input.rotation_offset || 0;
		/* END LAB 3 CHANGE */
		
		/*
			LAB 3: We want to store the result from above as a property of "data."
			This will come through as "input" the next time this server code is called from the client.
		*/
		data.rotation_offset =  data.options.chartRotation;
		/* END LAB 3 CHANGE */
		
		// Chart circumference, default 100. Min 25, max 100.
		/*
			LAB 3: Note that we are adding a check on "input."
			We are also storing the result to the "data" object to be used as "input" from the client.
			Also note that when we set the "data.options" property, we are using what we determined for data.circ as the value.
		*/
		data.options.circ = options.arc_percent || input.arc_percent;
		data.arc_percent = data.options.circ;
		data.options.chartCircumference = parseInt(data.options.circ) || 100;
		/* END LAB 3 CHANGE */
		
		if (isNaN(parseInt(data.options.chartCircumference)) || data.options.chartCircumference < 25 || data.options.chartCircumference > 100){
			data.options.chartCircumference = 100;
		}

		/*
			LAB 3: We want to store the title to the "data" object.
			Again, this will be passed in as "input" from the client script.
		*/
		data.title = options.title || input.title;

		/*
			LAB 3: Execute asynchronously.
			If we do not have "input" defined, then this is not being called from the client and we should just return.
		*/
		if (!input){
			return;
		}

		// If we are executing asynchronously, let's see what "data" looks like in the console.
		console.log(data);
		/* END LAB 3 CHANGE */
		
		// We can use the built-in chart colors settings. This allows you to define colors per table to be used in this or other out-of-the-box rerports
		function getColors(tbl,fld){
			var retArr = [];
			var g_colors = new GlideChartFieldColors(tbl,fld);
			data.chartLabels.forEach(function(lbl){
				var val = data.chartLookup[lbl];
				var col = val.split('=');
				col = (col.length > 1 ? col[1] : col[0]).toString();
				retArr.push(g_colors.get(col));
			});

			console.log(retArr);

			return retArr;
		}

		// If we have defined a table and field for this widget in the instance options, get the data to build the chart
		if (!!table && !!field){

			/*
				LAB 3: We have added a check on "input" for asynchronous execution.
			*/
			data.options.chartTitle = options.title || input.title || $sp.getValue('title');
			/* END LAB 3 CHANGE */
			
			data.options.table = table;
			data.options.display_field = field;
			data.options.filter = filter;

			var ga = new GlideAggregate(table);
			ga.addAggregate('COUNT',field);
			ga.orderByAggregate('COUNT',field);
			ga.addEncodedQuery(filter);
			ga.query();

			var oth = 0;
			var othItems = [];

			// We are setting a hard limit of 12 items for an "other" group. How might you make this configurable?
			while (ga.next() && oth < 12){
				var disp = !!ga.getDisplayValue(field) ? ga.getDisplayValue(field) : '(Empty)';
				data.chartLabels.push(disp);
				data.chartLookup[disp] = '=' + ga.getValue(field);
				othItems.push(ga.getValue(field));
				data.chartData.push(ga.getAggregate('COUNT',field));
				oth++;
			}

			// If we have reached our threshold for "Other," then our query should be anything that does not match what we have used so far
			if (oth == 12){
				var gr = new GlideRecord(table);
				gr.addEncodedQuery(filter);
				gr.addQuery(field,'NOT IN',othItems.toString());
				gr.query();

				if (gr.getRowCount() > 0){
					data.chartLabels.push('Other');
					data.chartLookup['Other'] = 'NOT IN' + othItems;
					data.chartData.push(gr.getRowCount());
				}
			}

			data.chartColors = getColors(table,field);

		}

	})();
    ```
    2. Client controller:
    ```javascript
	function(spUtil,$scope,$rootScope,$location) {
		/* widget controller */
		var c = this;

		/*
			LAB 3: Make this widget async.
			Calling c.server.update will call the server script and pass in our "data" object as "input."
			This will refresh our "data" object, this time with data to render in the chart.
		*/
		c.server.update();
		/* END LAB 3 CHANGES */
		
		// Here we define how the chart legend is displayed. How might you make this configurable?
		c.data.chartLegend = {
			display:true,
			position:'bottom',
			labels:{
				boxWidth:15
			}
		};

		// Here we are building our chart options. This controls the visual display and animations for the chart.
		c.data.chartOptions = {
			rotation:((-0.5 + (-c.data.options.chartRotation)/50)*Math.PI),
			circumference:2*(c.data.options.chartCircumference/100)*Math.PI,
			legend:c.data.chartLegend,
			animation:{
				duration: c.data.options.animate,
				easing:'easeOutBounce'
			},
			cutoutPercentage:c.data.options.donut_cutout
		};

		// If we have a lot of text in our legend labels, decrease the font size for better display.
		if (c.data.chartLabels.toString().length > 250){
			c.data.chartLegend.labels.fontSize = 10;
		}

		// Here we define what happens when you click on the chart. The points argument contains information about the area that was clicked, and the evt argument contains information about the event itself. 
		c.onClick = function (points, evt) {

			// Check to see if we have anything in the points array
			if (points.length > 0){
				// The first member of the points array contains information about the portion of the chart we clicked. The _model property does not contain a "value," just a label which is why we built the "chartLookup" object to index based on display value.  
				var filt = (!!c.data.options.filter ? c.data.options.filter + '^' : '') + c.data.options.display_field + c.data.chartLookup[points[0]._model.label];

				/*
					LAB 3: If the widget options tell us we are event-driven, then emit an event.
					We will rely on other widgets to "catch" and process this event.
					We do this instead of navigating away from the page.
				*/
				if (c.data.options.event_driven == 'true'){
					$scope.$emit('chart.click',{filter: filt});
				} else {
					// If not, then navigate like we did before.
					/* END LAB 3 CHANGES */
					
					// Now we build an object for the click target. This will be used to add parameters to the URL location
					var targObj = {
						id: c.data.targetPage,
						table:c.data.options.table,
						filter:filt,
						previous_id:c.data.currentPage,
						view:'sp'
					};
					$location.search(targObj);
				/*
					LAB 3: We need to close our newly introduced "IF/ELSE" block.
				*/
				}
				/* END LAB 3 CHANGES */
			}

		};

		// We want to watch for any changes, so we will use the recordWatch functionality to keep the chart up to date
		if (!!c.data.options.table){
			spUtil.recordWatch($scope,c.data.options.table,c.data.options.filter);
		}

	}
    ```
    3. CSS:
    ```css
    div.chartContainer {

      max-width: 500px;
      /*
          LAB 3: We are removing the "padding" from this CSS.
      */
      //padding: 8px;
      /* END LAB 3 CHANGES */
	  
      margin: auto !important;

    }
    ```
    4. Option schema (we are adding an option called Event Driven to the schema): 
    *(Note: this can also be updated in the Widget Editor using the Edit Option Schema as done in LAB 1)*
    ```javascript
    [
		{
			"hint":"Size of the \"donut hole,\" default is 50%",
			"name":"donut_cutout_percent",
			"section":"Presentation",
			"label":"Donut Cutout Percent",
			"type":"integer"
		},
		{
			"hint":"Percent of a circle the chart should consume. Min 25%",
			"name":"arc_percent",
			"section":"Presentation",
			"label":"Arc Percent",
			"type":"integer"
		},
		{
			"hint":"Percentage to rotate the chart from 0 degrees vertical",
			"name":"rotation_offset",
			"section":"Presentation",
			"label":"Rotation Offset",
			"type":"integer"
		},
		{
			"hint":"Number of milliseconds it takes to animate the chart",
			"name":"animation_speed",
			"section":"Presentation",
			"label":"Animation Speed",
			"type":"choice",
			"choices":[
				{
					"label":"0",
					"value":"0"
				},
				{
					"label":"500",
					"value":"500"
				},
				{
					"label":"750",
					"value":"750"
				},
				{
					"label":"1000",
					"value":"1000"
				},
				{
					"label":"1250",
					"value":"1250"
				},
				{
					"label":"1500",
					"value":"1500"
				}
			]
		},
		{
			"hint":"Identify the page to target when the chart is clicked. Default is \"list\"",
			"name":"target_page",
			"section":"Behavior",
			"label":"Target Page",
			"type":"string"
		},
		/*
			LAB 3: New instance option "Event-Driven"
			Allows you to select whether you want the widget to emit an event when the chart is clicked or to navigate to another page.
		*/
		{
			"hint":"Indicate whether you want to emit an event when the chart is clicked or navigate to another page",
			"name":"event_driven",
			"section":"Behavior",
			"label":"Event Driven",
			"type":"boolean"
		}
		/* END LAB 3 CHANGES */
	]
    ```
1. Navigate to **Service Portal** > **Widgets**
1. Open the **Combined Donut and Table** widget you created previously
1. Update the form as follows:
    1. Body HTML template:
    ```html
	<div>
	  <div class="alert alert-danger" ng-if="data.invalid_table">
		${Table not defined} '{{data.table_label}}'
	  </div>
	  <!-- Toggle between chart and table view. The toggle will set the value of the showTable variable to true or false when clicked. Notice that we are binding the watchVar scope property in our directive to the current scope's showTable property through the watch-var attribute. -->
	  <toggle-switch watch-var="showTable" title="Chart or Table" label-true="Table" label-false="Chart"></toggle-switch>
	  <!-- If showTable is false, show the chart. Note the use of ng-if here. The angular chartJS code checks on the chart at regular intervals.
			If the chart canvas is hidden when that check is performed there is an error and the next time you try to show the chart it will not render correctly. -->
	  <div ng-if="!showTable">
		<sp-widget ng-if="!!data.donutWidget" widget="data.donutWidget"></sp-widget>
	  </div>

	  <!-- If showTable is true, then show the data table widget. Note that it is okay to use ng-show here,
		as there is no requirement that the data table is visible for other processes. -->
	  <div ng-show="showTable">
		<sp-widget ng-if="data.dataTableWidget" widget="data.dataTableWidget"></sp-widget>
	  </div>

	  <!-- LAB 3: If we have a form widget, embed it. We will instantiate a widget if we click on the data table. -->
	  <sp-widget ng-if="!!formWidget" widget="formWidget"></sp-widget>
	  <!-- END LAB 3 CHANGES -->

	</div>
	```
    2. Client Controller:
    ```javascript
	function ($scope, spUtil, $location, spAriaFocusManager) {
		/* This is the original code copied from the data table widget */
		/*
			LAB 3: We are going to remove the original code for the data_table.click event.
			We will redefine what happens when the table is clicked.
		*/
		/*
		$scope.$on('data_table.click', function(e, parms){
			var p = $scope.data.page_id || 'form';
			var s = {id: p, table: parms.table, sys_id: parms.sys_id, view: 'sp'};
			var newURL = $location.search(s);
			spAriaFocusManager.navigateToLink(newURL.url());
		});
		*/
		/* END LAB 3 CHANGES */

		/* Here is our code. We are just defining the "showTable" property for $scope. Defaulting to false. */
		$scope.showTable = false;

		/*
			LAB 3: Everything from here to the end is new code.
			1. We are going to "watch" for the "chart.click" event emitted by the embedded chart widget.
			2. We are going to define a "formWidget" property for our current scope. Note that we modified the HTML in the previous step to check for this property.
			3. We are going to redefine the "watch" on our data_table.click" event.
			4. We are going to "watch" for an embedded modal's "closed" event.
		*/
		// Watch for the chart click event.
		$scope.$on('chart.click',function(evt,parms){
			console.log(parms);
			// Toggle showTable to true to show the table
			$scope.showTable = true;
			/*
				Let the table know about the updated filter.
				Notice that we are "broadcasting" this "away" from rootScope.
				This means that we are only notifying the data table that is embedded in this widget (or any of its descendants)
				If we have multiple instance of this widget on the page (hint-hint!), then this event will not be visible to the other embedded tables.
				The "setFilter" event is part of the out-of-the-box data table widget code.
			*/
			$scope.$broadcast('data_table.setFilter',parms.filter);
		});

		// When the table is clicked, show the target record in a form modal
		// Define the formWidget property here, manage it elsewhere.
		$scope.formWidget = '';

		/*
			Here we are changing what heppens when the data table is clicked.
			This event is emitted (sent "toward" rootScope) by the out-of-the box data table widget.
		*/
		$scope.$on('data_table.click',function(evt,parms){
			// Build params to get a form widget
			var formOpts = {
				embeddedWidgetId: 'widget-form',
				embeddedWidgetOptions: {
					table: parms.table,
					sys_id: parms.sys_id,
					view: 'sp',
					disableUIActions: 'true',
					hideRelatedLists: true
				}
			};

			/*
				Get our widget.
				spUtil.get returns a "promise." This means it executes asynchronously and will "do something" when the promise "resolves" (is finished).
				In our case, the result returned from a successful promise is a widget object that can be embedded.
				When the promise resolves, we will set the formWidget property of our instance's scope to the result.
				Note that we are not handling any promise failure. How would you do this?
			*/
			spUtil.get('widget-modal',formOpts).then(function(resp){
				$scope.formWidget = resp;
			});
		});

		/*
			Watch for the modal widget being closed.
			When the modal is closed, we will clear the formWidget property to make the HTML happy.
		*/
		$scope.$on('sp.widget-modal.closed',function(){
			$scope.formWidget = '';
		});
		/* END LAB 3 CHANGES */
	}
	```
    3. Option schema (we are adding an option called Event Driven to the schema): 
    *(Note: this can also be updated in the Widget Editor using the Edit Option Schema as done in LAB 1)*
    ```javascript
	[
		{
			"hint":"Size of the \"donut hole,\" default is 50%",
			"name":"donut_cutout_percent",
			"section":"Presentation",
			"label":"Donut Cutout Percent",
			"type":"integer"
		},
		{
			"hint":"Percent of a circle the chart should consume. Min 25%",
			"name":"arc_percent",
			"section":"Presentation",
			"label":"Arc Percent",
			"type":"integer"
		},
		{
			"hint":"Percentage to rotate the chart from 0 degrees vertical",
			"name":"rotation_offset",
			"section":"Presentation",
			"label":"Rotation Offset",
			"type":"integer"
		},
		{
			"hint":"Number of milliseconds it takes to animate the chart",
			"name":"animation_speed",
			"section":"Presentation",
			"label":"Animation Speed",
			"type":"choice",
			"choices":[
				{
					"label":"0",
					"value":"0"
				},
				{
					"label":"500",
					"value":"500"
				},
				{
					"label":"750",
					"value":"750"
				},
				{
					"label":"1000",
					"value":"1000"
				},
				{
					"label":"1250",
					"value":"1250"
				},
				{
					"label":"1500",
					"value":"1500"
				}
			]
		},
		{
			"hint":"Identify the page to target when the chart is clicked. Default is \"list\"",
			"name":"target_page",
			"section":"Behavior",
			"label":"Target Page",
			"type":"string"
		},
		/*
			LAB 3: This is our new instance option.
		*/
		{
			"hint":"Indicate whether you want to emit an event when the chart is clicked or navigate to another page",
			"name":"event_driven",
			"section":"Behavior",
			"label":"Event Driven",
			"type":"boolean"
		}
		/* END LAB 3 CHANGES */
	]
    ```
1. Navigate to **Service Portal** > **Pages**
1. Location and open the **CMDB Dashboard** page you created previously
1. Under Related Links, select **Open in Designer**
1. Click the **Edit** icon (![Edit Widget Icon](/images/edit_widget_v1.jpg)) in the upper right-hand corner of the widget
1. Locate the **Event Driven** option and check the box
1. Select **Save**
## Try it out!
1. Navigate back to the **CMDB Dashboard** page (the url should be https://[your-instance-url]/sp?id=k18_cmdb_dashboard). If you already have it open in another tab or window, refresh the page
1. When you click a section of the chart, the toggle should switch to **Table** and the data table should be visible and filtered to correspond to the portion of the chart that you clicked
1. When you click an item in the table, the corresponding record should open in a modal window

Congratulations, your widgets are now excellent communicators!
# Lab 4: Finally, Building the Single-Page Application
So now we have a reusable widget that contains other widgets that talk to each other. We have eliminated URL navigation from our page. What else can we do with this?
## Lab Goal
Understand **Angular Providers** and how to incorporate their functionality into your widget. Use a **Service** as an alternative means to facilitate communication between objects (scopes) on the page.
## But Wait! This is a *CMDB* Dashboard...
...not just a *Windows Server* dashboard. We need more widgets!
1. Navigate to **Service Portal** > **Pages**
1. Locate and open the **CMDB Dashboard** page you created previously
1. Under the Related Links section, select **Open in Designer**
1. Click the **Edit** icon (![Edit Widget Icon](/images/edit_widget_v1.jpg)) in the upper right-hand corner of the existing widget
1. In the window that appears, update the following:
    1. Table: **Servers [cmdb_ci_server]**
    1. Title: **All Servers**
1. Select **Save**
1. In the filter start typing: **Combined Donut and Table**
1. Drag and drop the **Combined Donut and Table widget** at the bottom of the page
1. Click the **Edit** icon (![Edit Widget Icon](/images/edit_widget_v1.jpg)) in the upper right-hand corner of the widget
1. In the window that appears, fill in the following:
    1. Table: **Network Gear [cmdb_ci_netgear]**
    1. Fields: **Name, IP Address, Description, Location, Operational Status**
    1. Maximum entries: **10**
    1. Display field: **Location**
    1. Order by: **Name**
    1. Title: **Network Gear**
    1. Donut Cutout Percent: **50**
    1. Arc Percent: **100**
    1. Rotation Offset: **0**
    1. Animation Speed:  **1250**
    1. Event Driven: **true** (checked)
1. Select **Save**
1. Drag and drop the **Combined Donut and Table** widget to the bottom of the page
1. Click the **Edit** icon (![Edit Widget Icon](/images/edit_widget_v1.jpg)) in the upper right-hand corner of the widget
1. In the window that appears, fill in the following:
    1. Table: Application **[cmdb_ci_appl]**
    1. Fields: **Name, Description, Operational Status, Version, Class**
    1. Maximum entries: **10**
    1. Display field: **Class**
    1. Order by: **Name**
    1. Title: **Applications**
    1. Donut Cutout Percent: **90**
    1. Arc Percent: **100**
    1. Rotation Offset: **0**
    1. Animation Speed:  **1500**
    1. Event Driven: **true** (checked)
1. Select **Save**
1. Drag and drop the **Combined Donut and Table** widget to the bottom of the page
1. Click the **Edit** icon (![Edit Widget Icon](/images/edit_widget_v1.jpg)) in the upper right-hand corner of the widget
1. In the window that appears, fill in the following:
    1. Table: **Business Service [cmdb_ci_service]**
    1. Fields: **Name, Description, Operational Status, Business criticality**
    1. Maximum entries: **10**
    1. Display field: **Business Criticality**
    1. Order by: **Name**
    1. Title: **Business Services**
    1. Donut Cutout Percent: **0**
    1. Arc Percent: **100**
    1. Rotation Offset: **0**
    1. Animation Speed:  **500**
    1. Event Driven: **true** (checked)
1. Select **Save**
## That's nice, but I *HATE* Scrolling. Bring on the Tabs!
Now that we have a lot of good information on our page, we need to make it more presentable. What if we could show one of the widgets at a time and use navigation to browse through them? Well, you know what we're about to do...
1. Navigate to **Service Portal** > **Angular Providers**
1. Select **New**
1. Fill out the form as follows:
    1. Type: **Service**
    2. Name: **k18PageManager**
    3. Client Script:
    ```javascript
    function(){

        // Keep track of all the components on the page
        this.pageComponents = {};

        // Provide an option for adding to the page components
        this.registerComponent = function(cObj){
            console.log(cObj);
            this.pageComponents[cObj.rectangle_id] = cObj;
            console.log(this.pageComponents);
        };

        // Get the current component
        this.currentComponent = '';

        this.setCurrent = function(cName){
            this.currentComponent = cName;
        };

        this.reset = function(){
            this.pageComponents = {};
            this.currentComponent = '';
        };

    }
    ```
1. Select **Submit**
1. Navigate to **Service Portal** > **Widgets**
1. Select **New**
1. Fill in the form as follows:
    1. Name: **Page Governor**
    2. Body HTML template:
    ```html
    <div>
      <p ng-if="!c.pm.currentComponent">No Components</p>
      <ul class="nav nav-tabs">
        <li ng-class="{'active': comp.rectangle_id == c.pm.currentComponent}" ng-repeat="comp in c.pm.pageComponents" ng-click="c.pm.setCurrent(comp.rectangle_id);"><a href="javascript: void(0)">{{ comp.data.title }} <span class="badge" ng-if="!!comp.data.recordCount">{{  comp.data.recordCount  }}</span></a></li>
      </ul>
    </div>
    ```
    3. CSS
    ```css
    .nav-tabs > li > a {
          border: 1px solid #ddd !important;
          background-color: #e6e6e6;
    }

    .nav-tabs > li > a:hover, .nav-tabs > li > a:focus {
         background-color: #fff;
    }

    .nav-tabs > li.active > a, .nav-tabs > li.active > a:focus {
     background-color: #fff; 
    }

    .nav-tabs > li.active > a:hover {
      background-color: lightGray;
    }
    .badge {
      background-color: darkGray;
    }
    ```
    4. Client controller
    ```javascript
    function(k18PageManager,$scope) {
      /* widget controller */
      var c = this;

        c.pm = k18PageManager;

        // reset on page load
        c.pm.reset();

        $scope.pc = c.pm.pageComponents;

        $scope.$watch('pc',function(newVal,oldVal){
            // If we do not have a current component, set it to the first one.
            if (Object.keys(c.pm.pageComponents).length > 0 && !c.pm.currentComponent){
                var currKey = Object.keys(c.pm.pageComponents)[0];
                c.pm.setCurrent(currKey);
            }
        },true);

    }
    ```
1. Select **Submit**
1. Search for and open the **Page Governor** widget you just created
1. Under the Angular Providers related list, select **Edit**
1. Locate the **k18PageManager** dependency in the “Collection” container (left)
1. Add it to the “Angular Providers List” container (right)
1. Select **Save**
## We Have Tabs, Now Let's Corral Them Widgets!
1. Navigate to **Service Portal** > **Widgets**
1. Locate and open the **Combined Donut and Table** widget you created previously
1. Update the form as follows:
    1. Body HTML template:
    ```html
    <div ng-show="c.widget.rectangle_id == c.pm.currentComponent">
      <div class="alert alert-danger" ng-if="data.invalid_table">
        ${Table not defined} '{{data.table_label}}'
      </div>
      <!-- Toggle between chart and table view. The toggle will set the value of the showTable variable to true or false when clicked. -->
      <toggle-switch watch-var="showTable" title="Chart or Table" label-true="Table" label-false="Chart"></toggle-switch>
      <!-- If showTable is false, show the chart. Note the use of ng-if here. The angular chartJS code checks on the chart at regular intervals.
    If the chart canvas is hidden when that check is performed there is an error and the next time you try to show the chart it will not render correctly. -->
      <div ng-if="showTable == false && c.widget.rectangle_id == c.pm.currentComponent">
        <sp-widget ng-if="!!data.donutWidget" widget="data.donutWidget"></sp-widget>
      </div>

      <!-- If showTable is true, then show the data table widget. Note that it is okay to use ng-show here,
    as there is no requirement that the data table is visible for other processes. -->
      <div ng-show="showTable == true">
        <sp-widget ng-if="data.dataTableWidget" widget="data.dataTableWidget"></sp-widget>
      </div>

      <sp-widget ng-if="!!formWidget" widget="formWidget"></sp-widget>

    </div>
    ```
    2. Client Controller:
    ```javascript
    function ($scope, spUtil, $location, spAriaFocusManager, k18PageManager,$http) {
        /*$scope.$on('data_table.click', function(e, parms){
            var p = $scope.data.page_id || 'form';
            var s = {id: p, table: parms.table, sys_id: parms.sys_id, view: 'sp'};
            var newURL = $location.search(s);
            spAriaFocusManager.navigateToLink(newURL.url());
        });*/

        var c = this;

        // Get record count to send up to the tabs
        $scope.data.recordCount = 0;

        c.pm = k18PageManager;

        // Register ourselves with the Page Manager service.
        k18PageManager.registerComponent(this.widget);

        $scope.showTable = false;

        function getCount(flt){
            // Told you we would use web services!
            $http.get('/api/now/table/' + $scope.data.table + '?sysparm_query=' + flt + '&sysparm_fields=sys_id').then(function(resp){
                $scope.data.recordCount = resp.data.result.length;
            });

        }

        getCount($scope.data.filter);

        // Watch for the chart click event.
        $scope.$on('chart.click',function(evt,parms){
            console.log(parms);
            // Toggle showTable to true to show the table
            $scope.showTable = true;
            // Let the table know about the updated filter.
            $scope.$broadcast('data_table.setFilter',parms.filter);
        });

        // When the table is clicked, show the target record in a form modal

        $scope.formWidget = '';

        $scope.$on('data_table.click',function(evt,parms){
            // Build params to get a form widget
            var formOpts = {
                embeddedWidgetId: 'widget-form',
                embeddedWidgetOptions: {
                    table: parms.table,
                    sys_id: parms.sys_id,
                    view: 'sp',
                    disableUIActions: 'true',
                    hideRelatedLists: true
                }
            };

            // Get our widget
            spUtil.get('widget-modal',formOpts).then(function(resp){
                $scope.formWidget = resp;
            });
        });

        // Watch for the modal widget being closed
        $scope.$on('sp.widget-modal.closed',function(){
            $scope.formWidget = '';
        });

        $scope.$watch('showTable',function(newVal){
            console.log('showTable changed to ' + newVal);
        },true);
    }
    ```
1. Select **Update**
1. Locate and open the **Combined Donut and Table** widget you just updated *(note that instead of updating and navigating back, you can always just right-click in the header and select **Save** to stay on the record; but you probably already knew that)*
1. Under the Angular Providers related list, select **Edit**
1. Locate the **k18PageManager** dependency in the “Collection” container (left)
1. Add it to the “Angular Providers List” container (right)
1. Select **Save**
1. Navigate to **Service Portal** > **Pages**
1. Locate and open the **CMDB Dashboard** page you created previously
1. Under Related Links, select **Open in Designer**
1. Drag and drop a **Container** object at the very top of the page
1. Drag and drop a **"12" column-width** block into the container you just added
1. Start typing **Page Governor** into the *Filter Widget* field
1. Drag and drop the **Page Governor** widget into the column you just added
## Try it out!
1. Navigate back to the CMDB Dashboard page (the url should be https://[your-instance-url]/sp?id=k18_cmdb_dashboard). If you already have it open in another tab or window, refresh the page.
1. You should now have a fully functional single-page app!
1. Each tab will show its corresponding chart/table widget
1. Clicking on the chart will flip to the table and apply a filter corresponding to the slice you clicked
1. Clicking any item in the table will open the record in a popup modal window
1. Try it on your mobile device for good measure

**Congratulations! You just made something Totally Awesome!**
