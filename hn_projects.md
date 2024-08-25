## What is HN working on: A structured dataset

The latest [Ask HN: What are you working on](https://news.ycombinator.com/item?id=41342017) thread just dropped. And to give my own answer, building structured datasets!

<img width="1208" alt="image" src="https://github.com/user-attachments/assets/769bd9cc-acc0-4f3f-b818-660a01963505">

You can download the dataset as a CSV here: https://github.com/getomni-ai/datasets/blob/main/hn_projects_dataset.csv

Or query directly with SQL using the connection string included below.


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


## How friendly were the replies

Reply sentiment was judged on a 0 to 2 scale (with 0 being the most negative). The overall result was `1.57`, so largely positive.
```sql
select avg(reply_sentiment::float) from hn_projects_august
where reply_sentiment is not null;
```

### Sentiment by Category
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


### Sentiment by Open Source
The same query can be applies on the `is_open_source` classification with obvious results.

```sql
SELECT ROUND(CAST(AVG(reply_sentiment::float) AS numeric), 2) AS avg_sentiment, is_open_source
FROM hn_projects_august
WHERE reply_sentiment IS NOT NULL
GROUP BY is_open_source
ORDER BY 1 DESC;
```

<img width="419" alt="image" src="https://github.com/user-attachments/assets/84f4bc2d-4a95-45ed-b648-423b55a9f209">

### Sentiment by AI classificataion
Surprisingly for HN, the AI projects were favored slightly ahead of non AI projects: 
```
SELECT ROUND(CAST(AVG(reply_sentiment::float) AS numeric), 2) AS avg_sentiment, ai_project
FROM hn_projects_august
WHERE reply_sentiment IS NOT NULL
GROUP BY ai_project
ORDER BY 1 DESC;
```

<img width="376" alt="image" src="https://github.com/user-attachments/assets/11e23063-ac42-45b8-b20c-87702a43190f">


## Industry Level Breakdown

I started off by classifying the comments into ~25 different industries. This is a better classification for the `STARTUP` projects, as some of the `PERSONAL_PROJECT` comments don't really need an industry classifier. i.e. one guy said he was working on his car, which got the `AUTOMOTIVE` tag.

The number one project type was `SOFTWARE_DEVELOPMENT`, which boiled down to primarily dev tools.

```sql
select project_industry, count(*) from hn_projects_august group by 1 order by 2 desc;
```

<img width="395" alt="image" src="https://github.com/user-attachments/assets/db37bea9-d156-45c4-9973-be1761c2b2a6">

<br><br>

Software development also dominated in reply count:

```sql
select project_industry, sum(reply_count::int) reply_count from hn_projects_august group by 1 order by 2 desc;
```
<img width="443" alt="image" src="https://github.com/user-attachments/assets/8a661887-52e3-4ff3-8c52-881f000efa83">

## Most popular Industries

This one is a bit hard since HN doesn't display explicit upvotes. But since populatity is roughly determined by position on page, and because I scraped the comments sequentially, we can use the `id` as a proxy. Note this is a pretty fuzzy populatity score. 

```sql
with total_comments as (
    select count(*) as count from hn_projects_august
)
select
    project_industry,
    ROUND(AVG(((select count from total_comments)::int - id::int)), 0) popularity,
    ROUND(avg(reply_count::int), 2) reply_count
from hn_projects_august
group by 1 order by 2 desc;
```

<img width="720" alt="image" src="https://github.com/user-attachments/assets/b21fb2bf-1220-42d8-99ba-9e0d9d6f5aee">

### But it mostly works

Looking at the `one_liner` column sorted by `id`. The top post was the [DIY Bike Battery](https://news.ycombinator.com/item?id=41344737) which got placed in the AUTOMOTIVE category since there wasn't a better fit.

<img width="1022" alt="image" src="https://github.com/user-attachments/assets/cd7b2502-9f26-405d-9f5f-2965564f1c55">


## Still digging in

I've only been playing with the data for a couple hours, so still some interesting items I want to pull out. If anyone has some thoughts on new columns to add, just drop me a note! (tyler@getomni.ai)




