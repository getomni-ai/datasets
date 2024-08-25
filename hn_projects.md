## What is HN working on: A structured dataset

The latest [Ask HN: What are you working on](https://news.ycombinator.com/item?id=41342017) thread just dropped. And to give my own answer, building structured datasets!

<img width="1208" alt="image" src="https://github.com/user-attachments/assets/769bd9cc-acc0-4f3f-b818-660a01963505">

## Getting the initial data set

I wrote a quick scraper for the HN comments. Just pulling every top level comment along with its replies as a nested object.

This ended up pulling 642 top level comments with about 458 replies. I created a posrgres db with this original data set. The `replies` I just concatenated together in the order they came in (with an `indent` field to mark what level comment it was. Then stringified the json array and added it to the db. 

```js
{
    id: 99,
    created_at: '2024-08-25T18:19:28.756364+00:00',
    hn_user: 'spuds',
    comment: 'Helping others with their mental health (after my own struggles).Worked as a software dev/manager for a decade, went through workaholism, burnout, then alcoholism, depression, all that. Doing a ton better now, and taking some time off to write about what I went through and hopefully help out others going through the same thing some: https://depthsofrepair.com/',
    replies: `[{"commentText":"This is helpful to me (and many others, I'm sure) and I look forward to reading more. Subscribed.","commentUser":"nowami","indent":"1","children":[]}]`,
    reply_count: 1
}
```

## Creating the structured data

I did this using my own tool of course ([Omni](https://getomni.ai/)).

I pulled out the following values: 

1. `project_category` - Enum - PERSONAL_PROJECT, STARTUP, SELF_IMPROVEMENT, OTHER
2. `is_open_source` - Boolean
3. `github_link` - String
4. `project_industry` - Enum - SOFTWARE_DEVELOPMENT, HEALTHCARE, EDUCATION, TRANSPORTATION, etc.
5. `one_liner` - String - A one line pitch for the project
6. `tech_stack` - String[]
7. `reply_sentiment` - Num - Sentiment betwee 0 and 2 for the comment replies
8. `demo_link` - String
9. `ai_project` - Boolean

Example setup:

<img width="1084" alt="image" src="https://github.com/user-attachments/assets/b565d788-cdd8-437e-8c64-6ae3118e7067">

## Analyzing the results

All the results are stored in a Postgres DB. So we can write SQL for all the analyzis. And plug into existing tools like Metabase for some visualizations.

If you want to write some queries yourself, you can use the following credentials: 
```
HOST=
PORT=5432
DATABASE=
USER=
PASSWORD=
TABLE=
```
Note this is a temporary table with read only permissions. 


### What's the breakdown of project type
```sql
select project_category, count(*) from hn_projects_august group by 1 order by 2 desc;
```
<img width="361" alt="image" src="https://github.com/user-attachments/assets/e068599b-d661-4ada-8b8c-0d2ef15999a9">


### How friendly were the replies

Reply sentiment was judged on a 0 to 2 scale (with 0 being the most negative). The overall result was `1.57`, so largely positive.
```sql
select avg(reply_sentiment::float) from hn_projects_august
where reply_sentiment is not null;
```

How does that break down by the `project_category`. Do HN commenters favor personal projects over startups?

```sql
SELECT ROUND(CAST(AVG(reply_sentiment::float) AS numeric), 2) AS avg_sentiment, project_category
FROM hn_projects_august
WHERE reply_sentiment IS NOT NULL
GROUP BY project_category
ORDER BY 1 DESC;
```

The answer is yes! Commenters favor self improvement posts the most, and startups the least.

<img width="434" alt="image" src="https://github.com/user-attachments/assets/73673615-5c4d-41e4-ae0d-df2a08b2df0f">













