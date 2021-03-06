---
title: "Data Cleaning and Exploratory Analysis of the 2019 College Football Season Using Expected Points Added (EPA) Data"
author: "Chad Peltier"
date: "12/10/2019"
output: 
  html_document:
    keep_md: true
---
This project is intended to take college football play-by-play (PBP) data and produce expected points added (EPA) summaries for individual games (i.e. EPA box scores), season stats for offense and defense, and for skill players. 

EPA allows us to evaluate the magnitude of success for an individual play, and by extension, a player, side of the ball, or entire team. While success rate (another common advanced stat) only looks at the binary question of “was this play successful or not?” based on predefined goals given a particular down and distance, EPA allows you to not only say that a play was good, but also *how* good. 

Expected points themselves are derived from a statistical model for each down, distance, and field position. If a play does better than expected given the play’s down, distance, and field position, then it will have a positive EPA. You can find [more about EPA here](https://www.advancedfootballanalytics.com/index.php/home/stats/stats-explained/expected-points-and-epa-explained).

## Loading our dataset
First, I loaded the three necessary packages and then wrote a simple for loop to create a data frame of PBP EPA data for the entire 2019 college football season. EPA data is courtesy of the cfbscrapR package.

```r
library(tidyverse)
```

```
## -- Attaching packages ------------------------------------------------------------------- tidyverse 1.3.0 --
```

```
## v ggplot2 3.2.1     v purrr   0.3.3
## v tibble  2.1.3     v dplyr   0.8.3
## v tidyr   1.0.0     v stringr 1.4.0
## v readr   1.3.1     v forcats 0.4.0
```

```
## -- Conflicts ---------------------------------------------------------------------- tidyverse_conflicts() --
## x dplyr::filter() masks stats::filter()
## x dplyr::lag()    masks stats::lag()
```

```r
library(cfbscrapR)
```

```
## Warning: replacing previous import 'mgcv::multinom' by 'nnet::multinom' when
## loading 'cfbscrapR'
```

```r
library(ggimage)

##Pull season data from cfbscrapR
pbp_2019 <- data.frame()
for(i in 1:15){
  model <- cfb_pbp_data(year = 2019, season_type = "both", week = i, epa_wpa = TRUE)
  df <- data.frame(model)
  pbp_2019 <- bind_rows(pbp_2019, df)
}
```

## Tidying 
Next we'll need to do some tidying and add a few new variables. 

First we'll adjust the pre-set rush and pass variables. While there is a "play_type" variable that can be used for most things, that variable has more categories than we want (i.e. more than just rush or pass), including types of touchdowns, turnovers, sacks, safeties and completions/incompletions. So the rush/pass mutations below adjust the rush and pass variables by using stringr::str_detect to check for specific words mentioned in the "play_text" variable. For example, it's not clear whether play_type == "Fumble Recovery (Opponent)" is a rush or a pass, so we can use stringr::str_detect to check for "pass" or "run" to classify it in our new rush/pass variables. 

Second, we'll also add a number of other new variables for things we might want to analyze later, including: stuffed run rate (the rate of runs stopped at or behind the line of scrimmage), opportunity rate (the percentage of rushes that gain 4 or more yards -- i.e. the plays that the offensive line "does its job"), epa_successes (creates a binary good/bad play variable based on whether EPA is positive or negative), short rush attempts/success (running plays with 2 or fewer yards to go), and standard/passing downs (down/distance combinations where a team could theoretically rush or pass, or situations where passing is much more likely). 


```r
pbp_2019 <- pbp_2019 %>%
  rename(adjusted_yardline = adj_yd_line,
         offense = offense_play,
         defense = defense_play) %>%
  mutate(rz_play = ifelse((adjusted_yardline <= 20), 1, 0), 
         so_play = ifelse((adjusted_yardline <= 40 | play_type == "(Passing Touchdown) | (Rushing Touchdown"), 1, 0),
         pass = if_else(play_type == "Pass Reception" | play_type == "Passing Touchdown" |
                          play_type == "Sack" | play_type == "Pass Interception Return" |
                          play_type == "Pass Incompletion" | play_type == "Sack Touchdown" |
                          (play_type == "Safety" & str_detect(play_text, "sacked")) |
                          (play_type == "Fumble Recovery (Own)" & str_detect(play_text, "pass")) |
                          (play_type == "Fumble Recovery (Opponent)" & str_detect(play_text, "pass")), 1, 0),
         rush = ifelse(play_type == "Rush" | play_type == "Rushing Touchdown" | (play_type == "Safety" & str_detect(play_text, "run")) |
                         (play_type == "Fumble Recovery (Opponent)" & str_detect(play_text, "run")) | 
                         (play_type == "Fumble Recovery (Own)" & str_detect(play_text, "run")), 1, 0),
         rush_pass = if_else(rush == 1, "rush", 
                             if_else(pass == 1, "pass", "NA")),
         stuffed_run = ifelse((rush == 1 & yards_gained <=0), 1, 0),
         opp_rate_run = ifelse((rush == 1 & yards_gained >= 4), 1, 0),
         epa_success = ifelse((rush == 1 | pass == 1) & EPA >= 0, 1, 0),
         epa_explosive = if_else((rush == 1 & EPA >= 1.7917221), 1, 
                                 if_else((pass == 1 & EPA >= 2.4486338), 1, 0)),
         short_rush_attempt = ifelse(distance <= 2 & rush == 1, 1, 0),
         short_rush_success = ifelse(distance <= 2 & rush == 1 & yards_gained >= distance, 1, 0),
         std.down = ifelse(down == 1, 1,
                        ifelse(down == 2 & distance < 8, 1, 
                           ifelse(down == 3 & distance < 5, 1,
                                  ifelse(down == 4 & distance < 5, 1, 0)))),
         pass.down = ifelse(down == 2 & distance > 8, 1, 
                            ifelse(down == 3 & distance > 5, 1, 
                                   ifelse(down == 4 & distance > 5, 1, 0)))
)
```

### Adding team logos
Next we'll add a data frame for team logos that will be helpful for charts later on. These logos are in individual image files on my computer, courtesy of collegefootballdata.com.

This section will first read in the logo image file locations into a list. Next we'll extract the team name from the image file location using the stringr package. 

However, because we have image files for all college football teams but we are only interested in teams at the FBS level, we'll need to filter out all of the non-FBS schools. To do that, we'll use a csv file of FBS teams (again courtesy of collegefootballdata.com) and use an inner join to filter out the teams that aren't on that list:

```r
logos_list <- as.data.frame(list.files("C:/Users/chad.peltier/OneDrive - IHS Markit/Desktop/CFB/logos", pattern = "*.png", full.names = TRUE)) 
colnames(logos_list)[1] <- "logo"

logo_team <- as_tibble(str_split(logos_list$log, "C:/Users/chad.peltier/OneDrive - IHS Markit/Desktop/CFB/logos", simplify = TRUE))
```

```
## Warning: `as_tibble.matrix()` requires a matrix with column names or a `.name_repair` argument. Using compatibility `.name_repair`.
## This warning is displayed once per session.
```

```r
logo_team <- logo_team %>% 
    mutate(team = str_replace(V2, ".png", ""),
           team = str_replace(team, "/", "")) %>%
    select(team)

logo_team <- cbind(logo_team, logos_list)

teams <- read_csv("teams.csv")
```

```
## Parsed with column specification:
## cols(
##   id = col_double(),
##   school = col_character(),
##   mascot = col_character(),
##   abbreviation = col_character(),
##   alt_name1 = col_character(),
##   alt_name2 = col_character(),
##   alt_name3 = col_character(),
##   conference = col_character(),
##   division = col_character(),
##   color = col_character(),
##   alt_color = col_character(),
##   `logos[0]` = col_character(),
##   `logos[1]` = col_character()
## )
```

```r
teams <- teams %>%
    rename(team = school)

teams_logo <- logo_team %>%
    inner_join(teams, by = "team") %>%
    select(team, logo)
```

### Adding player name columns using stringr and regular expressions (regex)
Next we'll do some more cleaning, this time adding columns for running back, quarterback, and wide receiver player names. That will allow us to calculate EPA data for individual players and for position groups as a whole. 

This section again uses the stringr package, but now incorporates regular expressions (regex) to pull out the player names from the "play_text" variable. As you can see below, the running back (RB) names were much simpler to extract than the other two position groups. QB and WR names followed multiple formats in the play_text variable, so we needed to use several conditional (ifelse) stringr:str_extract statements in order to capture all of the possible name formats.

```r
# RB names 
pbp_2019 <- pbp_2019 %>%
    mutate(rush_player = ifelse(rush == 1, str_extract(play_text, "(.{0,25} )run |(.{0,25} )\\d{0,2} Yd Run"), NA)) %>%
    mutate(rush_player = str_remove(rush_player, " run | \\d+ Yd Run"))

# QB names 
pbp_2019 <- pbp_2019 %>%
    mutate(pass_player = ifelse(pass==1, str_extract(play_text, "pass from (.*?) \\(|(.{0,30} )pass |(.{0,30} )sacked|(.{0,30} )incomplete "), NA)) %>%
    mutate(pass_player = str_remove(pass_player, "pass | sacked| incomplete")) %>%
    mutate(pass_player = if_else(play_type == "Passing Touchdown", str_extract(play_text, "from(.+)"), pass_player),
          pass_player = str_remove(pass_player, "from "), 
          pass_player = str_remove(pass_player, "\\(.+\\)"),
          pass_player = str_remove(pass_player, " \\,"))

# WR names
pbp_2019 <- pbp_2019 %>%
    mutate(receiver_player = ifelse(pass==1, str_extract(play_text, "to (.+)"), NA)) %>%
    mutate(receiver_player = if_else(str_detect(play_text, "Yd pass"), str_extract(play_text, "(.+)\\d"), receiver_player)) %>%
    mutate(receiver_player = ifelse(play_type == "Sack", NA, receiver_player)) %>%
    mutate(receiver_player = str_remove(receiver_player, "to "),
           receiver_player = str_remove(receiver_player, "\\,.+"),
           receiver_player = str_remove(receiver_player, "for (.+)"),
           receiver_player = str_remove(receiver_player, "( \\d{1,2})"))
```

## Creating summary data frames with z-scores and percentiles
Now that data tidying is all done, we can produce EPA summary data frames. The code below will produce three data frames: box scores, offense season stats, and defense season stats. 

The code is mostly the same for each data frame, mostly differing by how the data is grouped (dplyr::group_by) -- either by offense, defense, or offense and defense (for the box scores). 

Each data frame calculates a number of summary stats, including average EPA per play, EPA success rate, rushing and passing average EPA per play, among others. For each stat we'll also calculate the stat's z-score and the z-score's percentile. Z-scores and percentiles allow for much easier analysis, because an average EPA of "0.036" doesn't tell us much (besides the fact that it is positive) without knowing how it compares with other average EPA values. 

The percentiles ("summary_stat_p") are particularly helpful here, as they are much easier to understand than even the number of standard deviations a team is from the mean of that statistic. You can essentially interpret these percentiles as "the probability that a random team has an EPA statistic lower than this." For example, Alabama's average offensive EPA percentile is 98.2%, meaning that a randomly selected team has a 98.2% probability of having an average offensive EPA lower than Alabama.

Finally, below each season summary section we'll join the team logos data frame with the season summary stats data frames to use for charts later on. 

```r
## box score stats
box_score_stats <- pbp_2019 %>%
  group_by(offense, defense) %>%
  filter(rush == 1 | pass == 1) %>%
  summarize(
    avg_epa = mean(EPA, na.rm=TRUE),
    avg_epa_z = NA,
    avg_epa_p = NA,
    epa_sr = mean(epa_success, na.rm=TRUE),
    epa_sr_z = NA,
    epa_sr_p = NA,
    epa_er = mean(epa_explosive, na.rm = TRUE),
    epa_er_z = NA, 
    epa_er_p = NA,
    avg_epa_rush = mean(EPA[rush == 1], na.rm=TRUE),
    avg_epa_rush_z = NA,
    avg_epa_rush_p = NA,
    epa_sr_rush = mean(epa_success[rush == 1], na.rm=TRUE),
    epa_sr_rush_z = NA,
    epa_sr_rush_p = NA,
    short_rush_epa = mean(epa_success[short_rush_attempt==1]),
    short_rush_epa_z = NA,
    short_rush_epa_p = NA,
    avg_epa_pass = mean(EPA[pass == 1], na.rm=TRUE),
    avg_epa_pass_z = NA,
    avg_epa_pass_p = NA,
    epa_sr_pass = mean(epa_success[pass == 1], na.rm=TRUE),
    epa_sr_pass_z = NA,
    epa_sr_pass_p = NA,
    avg_rz_epa = mean(EPA[rz_play == 1]),
    avg_rz_epa_z = NA,
    avg_rz_epa_p = NA,
    avg_rz_epa_sr = mean(epa_success[rz_play == 1]),
    avg_rz_epa_sr_z = NA,
    avg_rz_epa_sr_p = NA,
    std_down_epa = mean(EPA[std.down==1]),
    std_down_epa_z = NA,
    std_down_epa_p = NA,
    pass_down_epa = mean(EPA[pass.down==1]),
    pass_down_epa_z = NA,
    pass_down_epa_p = NA,
    scoring_opp_epa = mean(EPA[scoring_opp==1]),
    scoring_opp_epa_z = NA,
    scoring_opp_epa_p = NA,
    rz_epa = mean(EPA[rz_play==1]),
    rz_epa_z = NA,
    rz_epa_p = NA
    ) %>% ungroup()

box_score_stats <- box_score_stats %>% 
  mutate(
    avg_epa_z = scale(avg_epa),
    avg_epa_p = pnorm(avg_epa_z),
    epa_sr_z = scale(epa_sr),
    epa_sr_p = pnorm(epa_sr_z),
    epa_er_z = scale(epa_er),
    epa_er_p = pnorm(epa_er_z),
    avg_epa_rush_z = scale(avg_epa_rush),
    avg_epa_rush_p = pnorm(avg_epa_rush_z),
    epa_sr_rush_z = scale(epa_sr_rush),
    epa_sr_rush_p = pnorm(epa_sr_rush_z),
    short_rush_epa_z = scale(short_rush_epa),
    short_rush_epa_p = pnorm(short_rush_epa_z),
    avg_epa_pass_z = scale(avg_epa_pass),
    avg_epa_pass_p = pnorm(avg_epa_pass_z),
    epa_sr_pass_z = scale(epa_sr_pass),
    epa_sr_pass_p = pnorm(epa_sr_pass_z),
    avg_rz_epa_z = scale(avg_rz_epa),
    avg_rz_epa_p = pnorm(avg_rz_epa_z),
    avg_rz_epa_sr_z = scale(avg_rz_epa_sr),
    avg_rz_epa_sr_p = pnorm(avg_rz_epa_sr_z),
    std_down_epa_z = scale(std_down_epa),
    std_down_epa_p = pnorm(std_down_epa_z),
    pass_down_epa_z = scale(pass_down_epa),
    pass_down_epa_p = pnorm(pass_down_epa_z),
    scoring_opp_epa_z = scale(scoring_opp_epa),
    scoring_opp_epa_p = pnorm(scoring_opp_epa_z),
    rz_epa_z = scale(rz_epa),
    rz_epa_p = pnorm(rz_epa_z)
    )


## new all season stats - offense
season_stats_offense <- pbp_2019 %>%
  group_by(offense) %>%
  filter(rush == 1 | pass == 1) %>%
    summarize(
      avg_epa = mean(EPA, na.rm=TRUE),
      avg_epa_z = NA,
      avg_epa_p = NA,
      epa_sr = mean(epa_success, na.rm=TRUE),
      epa_sr_z = NA,
      epa_sr_p = NA,
      epa_er = mean(epa_explosive, na.rm = TRUE),
      epa_er_z = NA, 
      epa_er_p = NA,
      avg_epa_success = mean(EPA[epa_success==1]),
      avg_epa_explosive = mean(EPA[epa_explosive==1]),
      avg_epa_rush = mean(EPA[rush == 1], na.rm=TRUE),
      avg_epa_rush_z = NA,
      avg_epa_rush_p = NA,
      epa_sr_rush = mean(epa_success[rush == 1], na.rm=TRUE),
      epa_sr_rush_z = NA,
      epa_sr_rush_p = NA,
      short_rush_epa = mean(epa_success[short_rush_attempt==1]),
      short_rush_epa_z = NA,
      short_rush_epa_p = NA,
      avg_epa_pass = mean(EPA[pass == 1], na.rm=TRUE),
      avg_epa_pass_z = NA,
      avg_epa_pass_p = NA,
      epa_sr_pass = mean(epa_success[pass == 1], na.rm=TRUE),
      epa_sr_pass_z = NA,
      epa_sr_pass_p = NA,
      avg_rz_epa = mean(EPA[rz_play == 1]),
      avg_rz_epa_z = NA,
      avg_rz_epa_p = NA,
      avg_rz_epa_sr = mean(epa_success[rz_play == 1]),
      avg_rz_epa_sr_z = NA,
      avg_rz_epa_sr_p = NA,
      std_down_epa = mean(EPA[std.down==1]),
      std_down_epa_z = NA,
      std_down_epa_p = NA,
      pass_down_epa = mean(EPA[pass.down==1]),
      pass_down_epa_z = NA,
      pass_down_epa_p = NA,
      scoring_opp_epa = mean(EPA[scoring_opp==1]),
      scoring_opp_epa_z = NA,
      scoring_opp_epa_p = NA,
      rz_epa = mean(EPA[rz_play==1]),
      rz_epa_z = NA,
      rz_epa_p = NA
    ) %>% ungroup()


season_stats_offense <- season_stats_offense %>%
  mutate(
    avg_epa_z = scale(avg_epa),
    avg_epa_p = pnorm(avg_epa_z),
    epa_sr_z = scale(epa_sr),
    epa_sr_p = pnorm(epa_sr_z),
    epa_er_z = scale(epa_er),
    epa_er_p = pnorm(epa_er_z),
    avg_epa_rush_z = scale(avg_epa_rush),
    avg_epa_rush_p = pnorm(avg_epa_rush_z),
    epa_sr_rush_z = scale(epa_sr_rush),
    epa_sr_rush_p = pnorm(epa_sr_rush_z),
    short_rush_epa_z = scale(short_rush_epa),
    short_rush_epa_p = pnorm(short_rush_epa_z),
    avg_epa_pass_z = scale(avg_epa_pass),
    avg_epa_pass_p = pnorm(avg_epa_pass_z),
    epa_sr_pass_z = scale(epa_sr_pass),
    epa_sr_pass_p = pnorm(epa_sr_pass_z),
    avg_rz_epa_z = scale(avg_rz_epa),
    avg_rz_epa_p = pnorm(avg_rz_epa_z),
    avg_rz_epa_sr_z = scale(avg_rz_epa_sr),
    avg_rz_epa_sr_p = pnorm(avg_rz_epa_sr_z),
    std_down_epa_z = scale(std_down_epa),
    std_down_epa_p = pnorm(std_down_epa_z),
    pass_down_epa_z = scale(pass_down_epa),
    pass_down_epa_p = pnorm(pass_down_epa_z),
    scoring_opp_epa_z = scale(scoring_opp_epa),
    scoring_opp_epa_p = pnorm(scoring_opp_epa_z),
    rz_epa_z = scale(rz_epa),
    rz_epa_p = pnorm(rz_epa_z)
  )


season_stats_offense <- season_stats_offense %>%
  rename(team = offense) %>%
  left_join(teams_logo, by = "team") %>%
  filter(logo != is.na(logo))

## season stats - defense
season_stats_defense <- pbp_2019 %>%
  group_by(defense) %>%
  filter(rush == 1 | pass == 1) %>%
  summarize(
    avg_epa = mean(EPA, na.rm=TRUE),
    avg_epa_z = NA,
    avg_epa_p = NA,
    epa_sr = mean(epa_success, na.rm=TRUE),
    epa_sr_z = NA,
    epa_sr_p = NA,
    epa_er = mean(epa_explosive, na.rm = TRUE),
    epa_er_z = NA, 
    epa_er_p = NA,
    avg_epa_success = mean(EPA[epa_success==1]),
    avg_epa_explosive = mean(EPA[epa_explosive==1]),
    avg_epa_rush = mean(EPA[rush == 1], na.rm=TRUE),
    avg_epa_rush_z = NA,
    avg_epa_rush_p = NA,
    epa_sr_rush = mean(epa_success[rush == 1], na.rm=TRUE),
    epa_sr_rush_z = NA,
    epa_sr_rush_p = NA,
    short_rush_epa = mean(epa_success[short_rush_attempt==1]),
    short_rush_epa_z = NA,
    short_rush_epa_p = NA,
    avg_epa_pass = mean(EPA[pass == 1], na.rm=TRUE),
    avg_epa_pass_z = NA,
    avg_epa_pass_p = NA,
    epa_sr_pass = mean(epa_success[pass == 1], na.rm=TRUE),
    epa_sr_pass_z = NA,
    epa_sr_pass_p = NA,
    avg_rz_epa = mean(EPA[rz_play == 1]),
    avg_rz_epa_z = NA,
    avg_rz_epa_p = NA,
    avg_rz_epa_sr = mean(epa_success[rz_play == 1]),
    avg_rz_epa_sr_z = NA,
    avg_rz_epa_sr_p = NA,
    std_down_epa = mean(EPA[std.down==1]),
    std_down_epa_z = NA,
    std_down_epa_p = NA,
    pass_down_epa = mean(EPA[pass.down==1]),
    pass_down_epa_z = NA,
    pass_down_epa_p = NA,
    scoring_opp_epa = mean(EPA[scoring_opp==1]),
    scoring_opp_epa_z = NA,
    scoring_opp_epa_p = NA,
    rz_epa = mean(EPA[rz_play==1]),
    rz_epa_z = NA,
    rz_epa_p = NA
  ) %>% ungroup()


season_stats_defense <- season_stats_defense %>%
  mutate(
    avg_epa_z = scale(avg_epa),
    avg_epa_p = pnorm(avg_epa_z),
    epa_sr_z = scale(epa_sr),
    epa_sr_p = pnorm(epa_sr_z),
    epa_er_z = scale(epa_er),
    epa_er_p = pnorm(epa_er_z),
    avg_epa_rush_z = scale(avg_epa_rush),
    avg_epa_rush_p = pnorm(avg_epa_rush_z),
    epa_sr_rush_z = scale(epa_sr_rush),
    epa_sr_rush_p = pnorm(epa_sr_rush_z),
    short_rush_epa_z = scale(short_rush_epa),
    short_rush_epa_p = pnorm(short_rush_epa_z),
    avg_epa_pass_z = scale(avg_epa_pass),
    avg_epa_pass_p = pnorm(avg_epa_pass_z),
    epa_sr_pass_z = scale(epa_sr_pass),
    epa_sr_pass_p = pnorm(epa_sr_pass_z),
    avg_rz_epa_z = scale(avg_rz_epa),
    avg_rz_epa_p = pnorm(avg_rz_epa_z),
    avg_rz_epa_sr_z = scale(avg_rz_epa_sr),
    avg_rz_epa_sr_p = pnorm(avg_rz_epa_sr_z),
    std_down_epa_z = scale(std_down_epa),
    std_down_epa_p = pnorm(std_down_epa_z),
    pass_down_epa_z = scale(pass_down_epa),
    pass_down_epa_p = pnorm(pass_down_epa_z),
    scoring_opp_epa_z = scale(scoring_opp_epa),
    scoring_opp_epa_p = pnorm(scoring_opp_epa_z),
    rz_epa_z = scale(rz_epa),
    rz_epa_p = pnorm(rz_epa_z)
  )

season_stats_defense <- season_stats_defense %>%
    rename(team = defense) %>%
    left_join(teams_logo, by = "team") %>%
    filter(logo != is.na(logo))
```

### Creating summary data frames for skill players
Next we can make data frames for each skill position -- RBs, QBs, and WRs: 

```r
rusher_stats_19 <- pbp_2019 %>%
  group_by(offense, rush_player) %>%
  filter(rush_player != is.na(rush_player) & rush_player != "TEAM " & rush == 1 & (sum(rush) > 40)) %>%
  summarize(
    avg_epa = mean(EPA, na.rm=TRUE),
    epa_sr = mean(epa_success, na.rm=TRUE),
    plays = n()
  )%>% ungroup()

receiver_stats_19 <- pbp_2019 %>%
  group_by(offense, receiver_player) %>%
  filter(receiver_player != is.na(receiver_player) & receiver_player != "TEAM" & pass == 1 & sum(pass) >= 10) %>%
  summarize(
    avg_epa = mean(EPA, na.rm=TRUE),
    epa_sr = mean(epa_success, na.rm=TRUE),
    plays = sum(pass)
  )%>% ungroup()

passer_stats_19 <- pbp_2019 %>%
  group_by(offense, pass_player) %>%
  filter(pass_player != is.na(pass_player) & pass_player != "TEAM" & pass == 1 & sum(pass) > 40) %>%
  summarize(
    passes = sum(pass),
    avg_epa = mean(EPA, na.rm=TRUE),
    epa_sr = mean(epa_success, na.rm=TRUE)
  ) %>% ungroup()
```

## Charting team summary data
We can make a data frame that combines offense and defense season average EPA and EPA success rates for all teams. This adds in logos and will be important for making basic FBS team summary charts. 

```r
season_off_epa <- season_stats_offense %>%
    select(1,4,7) 
season_def_epa <- season_stats_defense %>%
  select(1,4,7) 

season_epa <- season_off_epa %>%
    full_join(season_def_epa, by = "team") %>%
    rename(avg_epa_p_off = avg_epa_p.x, avg_epa_p_def = avg_epa_p.y) %>%
    mutate(avg_epa_p_def = 1-avg_epa_p_def)

season_epa <- season_epa %>%
    inner_join(teams_logo, by = "team") %>%
    rename(epa_sr_p_off = epa_sr_p.x, epa_sr_p_def = epa_sr_p.y)

season_epa <- as_tibble(season_epa)
```

Using the data frame we made in the code block above, we can then easily make a summary chart that shows all FBS teams plotted by their offensive and defensive EPA percentiles. 


```r
ggplot(data = season_epa, aes(x = avg_epa_p_off, y = avg_epa_p_def)) +
    geom_image(aes(image = logo), size = .03, by = "width", asp = 1.8) +
    xlab("Offensive EPA per play") +
    ylab("Defensive EPA per play") +
    labs(caption = "Chart by Chad Peltier, EPA data from cfbscrapR, PBP data from @CFB_Data") +    ggsave("epa_off_def.png", height = 9/1.2, width = 16/1.2)
```

![](epa_analysis_files/figure-html/unnamed-chunk-8-1.png)<!-- -->

## Top teams and players by average EPA
A few more things we can do with this data. First, here are the top 5 offenses: 

```r
season_epa %>%
    arrange(desc(as.numeric(avg_epa_p_off))) %>%
    select(team, avg_epa_p_off) %>%
    head(5)
```

```
## # A tibble: 5 x 2
##   team       avg_epa_p_off[,1]
##   <chr>                  <dbl>
## 1 Alabama                0.982
## 2 LSU                    0.981
## 3 Oklahoma               0.975
## 4 Ohio State             0.970
## 5 Clemson                0.961
```

And defenses: 

```r
season_epa %>%
    arrange(desc(as.numeric(avg_epa_p_def))) %>%
    select(team, avg_epa_p_def) %>%
    head(5)
```

```
## # A tibble: 5 x 2
##   team            avg_epa_p_def[,1]
##   <chr>                       <dbl>
## 1 Clemson                     0.980
## 2 Ohio State                  0.969
## 3 Georgia                     0.933
## 4 San Diego State             0.924
## 5 UAB                         0.917
```
The first five offenses fit well with what we would expect based on other sources, such as the SP+ rankings. But for defensive EPA, while Clemson, Ohio State and Georgia also rank highly in other metrics, San Diego State and UAB only rank 17th and 28th respectively in the opponent-adjusted defensive SP+ ratings. That *could* suggest that opponent-adjusted EPA data would produce different results. 

Of course, we can't generalize the effectiveness of average EPA only by the top 5 results, but they are interesting nevertheless! 

Let's take a look at our top-5 quarterbacks, too: 

```r
passer_stats_19 %>%
    arrange(desc(avg_epa)) %>%
    head(5)
```

```
## # A tibble: 5 x 5
##   offense    pass_player       passes avg_epa epa_sr
##   <chr>      <chr>              <dbl>   <dbl>  <dbl>
## 1 Oklahoma   "Jalen Hurts "       287   0.493  0.502
## 2 Alabama    "Tua Tagovailoa "    237   0.458  0.439
## 3 Ohio State "Justin Fields "     275   0.412  0.447
## 4 Minnesota  "Tanner Morgan "     265   0.388  0.449
## 5 LSU        "Joe Burrow "        407   0.337  0.459
```
This seems to be a solid list of the top-5 FBS quarterbacks. Jalen Hurts, Justin Fields, and Joe Burrow were three of the four players selected as Heisman Finalists this year. Tua Tagovailoa was on many lists until he got injured. Tanner Morgan is a little surprising in the fourth spot, but has produced a relatively large number of explosive plays for Minnesota. 

## Visualizing all plays per game 
One final thing we can do is to visualize all plays for a specific team. Below is the EPA for every Ohio State offensive play this season, separated by game, with box plots to show the distribution of EPA:


```r
osu_plays <- pbp_2019 %>%
    filter(!is.na(rush_pass) & offense == "Ohio State") %>%
    mutate(defense = if_else(game_id == 401132983, "Wisconsin_2", defense))

osu_plays$defense <- factor(osu_plays$defense, levels = c("Florida Atlantic", "Cincinnati", "Indiana",
                                                      "Miami (OH)", "Nebraska", "Michigan State",
                                                      "Northwestern", "Wisconsin", "Maryland",
                                                      "Rutgers", "Penn State", "Michigan",
                                                      "Wisconsin_2"))
    

ggplot(data = osu_plays, aes(x = defense, y = EPA)) + 
    geom_boxplot() +
    geom_jitter(shape = 16, position = position_jitter(0.2)) +
ggsave("epa_game_osu.png", height = 9/1.2, width = 16/1.2)
```

![](epa_analysis_files/figure-html/unnamed-chunk-12-1.png)<!-- -->


And here's the same for defenses:

```r
osu_plays_def <- pbp_2019 %>%
    filter(!is.na(rush_pass) & defense == "Ohio State") %>%
    mutate(offense = if_else(game_id == 401132983, "Wisconsin_2", offense))

osu_plays_def$offense <- factor(osu_plays_def$offense, levels = c("Florida Atlantic", "Cincinnati", "Indiana",
                                                      "Miami (OH)", "Nebraska", "Michigan State",
                                                      "Northwestern", "Wisconsin", "Maryland",
                                                      "Rutgers", "Penn State", "Michigan",
                                                      "Wisconsin_2"))
    

ggplot(data = osu_plays_def, aes(x = offense, y = EPA)) + 
    geom_boxplot() +
    geom_jitter(shape = 16, position = position_jitter(0.2)) +
ggsave("epa_game_osu_def.png", height = 9/1.2, width = 16/1.2)
```

![](epa_analysis_files/figure-html/unnamed-chunk-13-1.png)<!-- -->







