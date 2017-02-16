# UBER Challenge
**Solution to the UBER Challenge**
*by Sergio Escobedo* 
# Part 1-SQL Syntax

##Excercise 1

1- For the last 30 days deduce the mean and median difference between Actual and Predicted ETA of all trips in the cities of ‘Qarth’ and ‘Meereen’

Assuming a PostgreSQL database, server timezone UTC

        select avg(actual_eta-predicted_eta),
               percentile_cont(0.5) WITHIN GROUP (ORDER BY actual_eta-predicted_eta)
        from trips a left join cities b on 
              a.city_id=b.city_id
        where b.city_name in ('Qarth','Meereen') 
          and DATE_PART('day', '2016-01-31 23:00:00' - request_at)<=30 
          and status='completed';

##Excercise 2

2- An event is logged in the events table with a timestamp each time a new rider attempts a sign up (with an event name 'attempted_sign_up') or successfully signs up (with an event name of 'sign_up_success'). For all riders signing up successfully in ‘Qarth’ and ‘Meereen’ in the first of week of 2016, find in each city for each day of the week, the percentage of riders who then complete a trip within 168 hours of the sign up date.

Assuming a PostgreSQL database, server timezone UTC and that a "rider" becomes a "driver" in the trips table

      select q1.city_name,
             case when q1.numofweek=1 then 'Mon'
                  when q1.numofweek=2 then 'Tue'
                  when q1.numofweek=3 then 'Wed'
                  when q1.numofweek=4 then 'Thu'
                  when q1.numofweek=5 then 'Fri'
                  when q1.numofweek=6 then 'Sat'
                  when q1.numofweek=7 then 'Sun'
             end as dayofweek,
             (sum(q2.drsupcomp)/sum(q1.alldrsup))*100 as percbecomedrivers 
      from       
          (select city_name,extract(dow from a._ts) numofweek,
                 count(distinct rider_id) alldrsup
           from events a left join cities b 
                        on
                        a.city_id=b.city_id
           where event_name='sign_up_success' and b.city_name in ('Qarth','Meereen') and extract(week from _ts)=1
           group by city_name,extract(dow from a._ts)
           order by 1,2 ) q1
      left join
          (select city_name,extract(dow from a._ts) as numofweek,
                 count(distinct rider_id) drsupcomp
          from events a left join cities b 
                        on a.city_id=b.city_id
                        left join trips c
                        on a.rider_id=c.driver_id
          where event_name='sign_up_success' and b.city_name in ('Qarth','Meereen') and extract(week from _ts)=1
                and c.status='completed' and 	DATE_PART('day', c.request_at - a._ts) * 24 + DATE_PART('hour', c.request_at -                                                           a._ts )<=168
          group by city_name,extract(dow from a._ts)
          order by 1,2) q2
      on q1.city_name=q2.city_name and q1.numofweek=q2.numofweek
      group by q1.city_name,
             case when q1.numofweek=1 then 'Mon'
                  when q1.numofweek=2 then 'Tue'
                  when q1.numofweek=3 then 'Wed'
                  when q1.numofweek=4 then 'Thu'
                  when q1.numofweek=5 then 'Fri'
                  when q1.numofweek=6 then 'Sat'
                  when q1.numofweek=7 then 'Sun'
             end;

# Part 2-Experiment and metrics design

   Answered in http://prezi.com/ae1zxixs9bma/?utm_campaign=share&utm_medium=copy 

# Part 3-Data analysis

        library(sqldf)
        library(dplyr)
        library(xtable)
        library(dplyr)
        library(rpart)
        library(rattle)
        library(rpart.plot)
        library(RColorBrewer)
        library(ggplot2)

        ####Analysis and Visualitation####

        #getting the data
        Ch_01<- read.csv(file="ds_challenge_v2_data_(4)_(1).csv", header=TRUE, sep=",")

        #formatting variables
        Ch_01$signup_date<-as.Date(Ch_01$signup_date,"%d/%m/%y")
        Ch_01$bgc_date<-as.Date(Ch_01$bgc_date,"%d/%m/%y")
        Ch_01$vehicle_added_date<-as.Date(Ch_01$vehicle_added_date,"%d/%m/%y")
        Ch_01$first_completed_date<-as.Date(Ch_01$first_completed_date,"%d/%m/%y")

        #indicator of complete trip
        Ch_01$complete<-ifelse(is.na(Ch_01$first_completed_date)==TRUE,0,1)
        Ch_01$complete<-factor(Ch_01$complete)

        #Calculation of auxiliar variables
        Ch_01$sign_bgc<-as.numeric(difftime(Ch_01$bgc_date ,Ch_01$signup_date, units = c("days")))
        Ch_01$bgc_vad<-as.numeric(difftime(Ch_01$vehicle_added_date ,Ch_01$bgc_date, units = c("days")))
        Ch_01$sign_vad<-as.numeric(difftime(Ch_01$vehicle_added_date ,Ch_01$signup_date, units = c("days")))
        Ch_01$sign_fcd<-as.numeric(difftime(Ch_01$first_completed_date ,Ch_01$signup_date, units = c("days")))
        Ch_01$complete<-factor(Ch_01$complete)
        Ch_01$quartile_sf <- ntile(Ch_01$sign_fcd,40) 

        head(Ch_01)
        tail(Ch_01)
        dim(Ch_01)
        names(Ch_01)
        str(Ch_01)
        summary(Ch_01)
        glimpse(Ch_01)


        hist(Ch_01$sign_bgc,main="Histogram of the time bt signup and bakground check")
        hist(Ch_01$bgc_vad,main="Histogram of the time bt background check and vehicle added")
        hist(Ch_01$sign_vad,main="Histogram of the time bt signup and vehicle added")
        hist(Ch_01$sign_fcd,main="Histogram of the time bt signup and first completed")

        boxplot(Ch_01$sign_bgc,main="Histogram of the time bt signup and bakground check")
        boxplot(Ch_01$bgc_vad,main="Histogram of the time bt background check and vehicle added")
        boxplot(Ch_01$sign_vad,main="Histogram of the time bt signup and vehicle added")
        boxplot(Ch_01$sign_fcd,main="Histogram of the time bt signup and first completed")



        boxplot(sign_fcd~city_name,data=Ch_01, main="Days between sign up and first completed ride",xlab="City",ylab="Days")
        boxplot(sign_vad~city_name,data=Ch_01, main="Days between sign up and first completed ride",xlab="City",ylab="Days")
        boxplot(sign_bgc~city_name,data=Ch_01, main="Days between sign up and first completed ride",xlab="City",ylab="Days")


        plot(Ch_01$city_name,Ch_01$complete)
        plot(Ch_01$signup_channel,Ch_01$signup_os)

        hist(Ch_01$signup_date, breaks=32)
        hist(Ch_01$complete)

        xtab = xtabs(~ city_name + quartile_sf , data=Ch_01)
        ptbl = prop.table(xtab, margin=1)
        ptbl
        #to searh for the limits of the quartiles
        table(Ch_01$quartile_sf,Ch_01$sign_fcd)

        #now we can se that quantile are like [1-5],(5-9],(9-14],(14-23],(23,30] so we can see 
        #that Berton and Strark has near of the 50% in the first 9 days vs Wrouver wich is 30%

        ####Cleansing####

        #looking the summary asumming a funnel of sign up cannot be possible to add a car when you dont even sign up in the system
        #so i decide to remove this case i mean is just one case

        Ch_01<-filter(Ch_01,Ch_01$sign_vad>0 | is.na(Ch_01$sign_vad)==TRUE |Ch_01$sign_vad==0 )



        ####Missingness####
        #i can infer that in our target variable wich is if the driver has been completed a trip, 
        #the missing data is an indicator for the drivers didn't has complete a trip so this is not a problem


        ####Modeling####
        set.seed(1)

        # Shuffle the dataset, call the result shuffled
        n <- nrow(Ch_01)
        shuffled <- Ch_01[sample(n),]

        # Split the data in train and test
        train_indices<-1:round(0.7*n)
        train<-shuffled[train_indices,]

        test_indices <- (round(0.7 * n) + 1):n
        test <- shuffled[test_indices, ]

        str(train)
        str(test)

        #tree<-rpart(complete~.,train,method="class", parms = list(split = "information"))
        tree<-rpart(complete~.,train,method="class", parms = list(split = "gini")) ###with gini chriterion
        pred<-predict(tree,test,type="class")

        conf<-table(test$complete,pred)

        sum(diag(conf))/sum(conf)

        prune<-prune(tree,cp=0.01)

        prune

        fancyRpartPlot(tree)
        fancyRpartPlot(prune)


        ####cross validation algorithm####
        accs <- rep(0,7)

        for (i in 1:7) {
          indices <- (((i-1) * round((1/7)*nrow(shuffled))) + 1):((i*round((1/7) * nrow(shuffled))))

          # Exclude them from the train set
          train <- shuffled[-indices,]

          # Include them in the test set
          test <- shuffled[indices,]

          # A model is learned using each training set
          tree <- rpart(complete ~ ., train, method = "class")

          # Make a prediction on the test set using tree
          pred<-predict(tree,test,type="class")

          # Assign the confusion matrix to conf
          conf<-table(test$complete,pred)

          # Assign the accuracy of this model to the ith index in accs
          accs[i] <- sum(diag(conf))/sum(conf)
        }

        # Print out the mean of accs
        mean(accs)



