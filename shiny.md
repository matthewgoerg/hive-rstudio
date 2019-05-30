# Dashboard with Shiny

The HQL code is not very exciting in this project because there is only one table and the 
column names are masked so it is difficult to formulate interesting analytic questions.

I will add the instructions to set up the next part soon, but it involves moving the data 
from the flat files to GCP's BigQuery. Once the data is there, we can use it to build a Shiny 
app.

```r
# This code makes the dashboard that can be seen here: # https://github.com/matthewgoerg/hive-rstudio/blob/master/dashboard.png
# Replace xxxxxx with the name of your GCP project.

## app.R ##
library(shiny) # loads Shiny
library(shinydashboard) # for special dashboard features in Shiny
library(tidyverse) # for data manipulation
library(RODBCext) # I forget
library(plotly) # enhances the ggplots with better graphics and interactivity
library(bigrquery) # allows R to communicate with GCP’s BigQuery

# my custom theme
theme_1 <- 	theme(legend.position="none", axis.text=element_text(size=11, face="bold"), 
                  strip.background = element_blank(), panel.background = element_rect(fill="white"),
                  panel.grid.major = element_line(colour = "lightgray"),
                  panel.grid.major.x = element_blank(),
                  panel.grid.minor.x = element_blank(),
                  panel.grid.minor.y = element_blank(),
                  axis.ticks = element_blank()) 

sidebar <- dashboardSidebar(
  sidebarMenu(
    menuItem("Referral", tabName = "referral", icon = icon("dashboard")),
    menuItem("Other", icon = icon("th"), tabName = "other",
             badgeLabel = "new", badgeColor = "green")
  )
)
body <- dashboardBody(
  tabItems(
    tabItem(tabName = "referral",
            # Boxes need to be put in a row (or column)
            fluidRow(
              box(plotlyOutput("plot1", height = 400)),
              box(plotlyOutput("plot2", height = 400))
            ),
            fluidRow(
              box(plotlyOutput("plot3", height = 400)),
              box(plotlyOutput("plot4", height = 400))
            )
    ),
    tabItem(tabName = "other",
            h2("Widgets tab content")
    )
  )
)


ui <- dashboardPage(
  dashboardHeader(title = "Referral website"),
  sidebar,
  body
)

server <- function(input, output) {
  totalHitsData <- reactive({
    
    project <- "xxxxxx" # put your project ID here
    
    sql <- "
    select 
      pn.print_name,
      CAST(sum(column10) AS float) / 1000000 as COUNT_ 
    from 
      [xxxxxx:test.click] l
    inner join
      [xxxxxx:test.print_names] pn ON l.column27 = pn.masked
    group by 
      pn.print_name 
    order by 
      COUNT_ desc
    "
    
    # Execute the query and store the result
    res <- query_exec(sql, project = project, useLegacySql = FALSE)
    
  })
  pctConversionData <- reactive({
    
    project <- "xxxxxxx" # put your project ID here
    
    sql <- "
    select 
      pn.print_name,
      CAST(sum(column10) AS float) / 1000000 as COUNT_,
      CAST(sum(column1) AS float) / count(column27) * 100 AS avg_clicks
    from 
      [xxxxxx:test.click] l
    inner join
      [xxxxxx:test.print_names] pn ON l.column27 = pn.masked
    group by 
      pn.print_name
    order by 
      COUNT_ desc
    "
    
    # Execute the query and store the result
    res <- query_exec(sql, project = project, useLegacySql = FALSE)
  })
  timeSpentData <- reactive({
    
    project <- "xxxxxx" # put your project ID here
    
    sql <- "
    select 
      pn.print_name,
      CAST(sum(column10) AS float) / 1000000 as COUNT_,
      CAST(sum(column10) AS float) / count(column27) AS avg_time
    from 
      [xxxxxx:test.click] l
    inner join
      [xxxxxx:test.print_names] pn ON l.column27 = pn.masked
    group by 
      pn.print_name
    order by 
      COUNT_ desc
    "
    
    # Execute the query and store the result
    res <- query_exec(sql, project = project, useLegacySql = FALSE)
  })
  timeSeriesData <- reactive({
    
    project <- "xxxxxx" # put your project ID here
    
    sql <- "
    select 
      pn.print_name,
      HOUR(time) AS HOUR_,
      COUNT(column27) AS COUNT_
    from 
      [xxxxxx:test.limited] l
    inner join
      [xxxxxx:test.print_names] pn ON l.column27 = pn.masked
    GROUP BY
      pn.print_name,
      HOUR_
    ORDER BY
      HOUR_
    "
    
    # Execute the query and store the result
    res <- query_exec(sql, project = project, useLegacySql = FALSE)
  })
  output$plot1 <- renderPlotly({
    ggplot(totalHitsData(), aes(reorder(pn_print_name, -COUNT_), COUNT_, fill = pn_print_name, text = COUNT_)) + 
      geom_bar(stat="identity") + theme_1 + xlab("referral website") +
      ylab("number of hits (in millions)") + ggtitle("Number of hits by referral website") 
  })
  output$plot2 <- renderPlotly({
    ggplot(pctConversionData(), aes(reorder(pn_print_name, -COUNT_), avg_clicks, fill = pn_print_name)) + 
      geom_bar(stat="identity") + theme_1 + xlab("referral website") +
      ylab("conversion rate (percent)") + ggtitle("Conversion rate by referral website")
  })
  output$plot3 <- renderPlotly({
    ggplot(timeSpentData(), aes(reorder(pn_print_name, -COUNT_), avg_time, fill = pn_print_name)) + 
      geom_bar(stat="identity") + theme_1 + xlab("referral website") +
      ylab("number of minutes spent on website") + ggtitle("Average time spent on website by referral website")
  })
  output$plot4 <- renderPlotly({
    ggplot(timeSeriesData(), aes(HOUR_, COUNT_, group = pn_print_name, color = pn_print_name)) + 
      geom_line(size = 1.5) + theme_1 + theme(legend.position = "right") + xlab("hour of the day") +
      ylab("number of hits by site (per hour)") + ggtitle("Number of hits over time by referral website")
  })
}

shinyApp(ui, server)
```




