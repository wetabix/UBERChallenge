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

# Part 3-Data analysis



