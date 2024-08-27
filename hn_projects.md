## What is HN working on: A structured dataset

The latest [Ask HN: What are you working on](https://news.ycombinator.com/item?id=41342017) thread just dropped. And to give my own answer, building structured datasets!

<img width="1208" alt="image" src="https://github.com/user-attachments/assets/769bd9cc-acc0-4f3f-b818-660a01963505">

You can download the dataset as a CSV here: https://github.com/getomni-ai/datasets/blob/main/hn_projects_dataset.csv

Or query directly with SQL using the connection string included below. Note this is a temporary table with read only permissions.

```
HOST=aws-0-us-east-1.pooler.supabase.com
PORT=6543
DATABASE=postgres
USER=postgres.raeysmhjbudociwvbwre
PASSWORD=!HZRdGLiiFC5iRj
TABLE=hn_projects_august
```

Full list of all the Open Source projects is [at the end](https://github.com/getomni-ai/datasets/blob/main/hn_projects.md#all-the-open-source-projects).


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
group by 1 order by 3 desc;
```

<img width="720" alt="image" src="https://github.com/user-attachments/assets/b21fb2bf-1220-42d8-99ba-9e0d9d6f5aee">

### But it mostly works

Looking at the `one_liner` column sorted by `id`. The top post was the [DIY Bike Battery](https://news.ycombinator.com/item?id=41344737) which got placed in the AUTOMOTIVE category since there wasn't a better fit.

<img width="1022" alt="image" src="https://github.com/user-attachments/assets/cd7b2502-9f26-405d-9f5f-2965564f1c55">


## Still digging in

I've only been playing with the data for a couple hours, so still some interesting items I want to pull out. If anyone has some thoughts on new columns to add, just drop me a note! (tyler@getomni.ai)

## All the open source projects

Here's a full dump of all the open source projects from the post. 

| github\_link | one\_liner | project\_category |
| :--- | :--- | :--- |
| https://github.com/mlang/mc1 | SuperCollider Reimagined Using Python and JIT Compilation | PERSONAL\_PROJECT |
| https://github.com/jonroig/usBabyNames.js | App Simplifies Baby Name Selection with Data Driven Filters | PERSONAL\_PROJECT |
| https://github.com/Pulselyre/UpbeatUI | Developing Pulselyre Touchfocused Windows App For Live Electronic Music | PERSONAL\_PROJECT |
| https://github.com/ziolko/eink-calendar-display | Designing Custom Hardware for Open Source SaaS Project | PERSONAL\_PROJECT |
| https://github.com/upvpn/upvpn-app | Modern Serverless VPN Explores App Store Publishing Process | STARTUP |
| https://github.com/incidentalhq/incidental | Open Source Incident Management Platform Seeking Early Feedback | PERSONAL\_PROJECT |
| https://github.com/GauntletWizard/cfssl/tree/ted/constraints | Bringing Enterprise-Grade Encryption and CA Infrastructure to Small Businesses | PERSONAL\_PROJECT |
| https://github.com/mikewarot/Bitgrid | Exploring GitHub Projects And Developing New Skills | PERSONAL\_PROJECT |
| https://github.com/hsnice16/golang\_learning | Transition to Full-Stack Development and Documenting GoLang Learning | SELF\_IMPROVEMENT |
| https://github.com/KaliedaRik/Scrawl-canvas | Maintaining and Improving My Canvas Library with New Filters | PERSONAL\_PROJECT |
| https://codeberg.org/treyd/ecksport | Developing a Protocol Library for Byte-Oriented Data Structures | PERSONAL\_PROJECT |
| https://github.com/dickeyy/passwords | Open Source Encrypted Password Manager Self Hosted And Customizable | PERSONAL\_PROJECT |
| https://github.com/cutestuff/FoodDepressionConundrum/blob/ma...latest | Probiotic Solution Eases Plant Digestion Challenges | PERSONAL\_PROJECT |
| https://github.com/MeoMix/symbiants | Colony Simulation Game in Rust for Personal Growth | PERSONAL\_PROJECT |
| https://github.com/anacrolix/possum | Possum: Efficient Disk-Backed Cache for File I/O and Snapshots | PERSONAL\_PROJECT |
| https://github.com/ubavic/mint | Development Update on Custom Markup Language Atex and Its Compiler | PERSONAL\_PROJECT |
| https://github.com/linkwarden/linkwarden | Self-Hostable Open-Source Collaborative Bookmark Manager Available | PERSONAL\_PROJECT |
| https://github.com/domino14 | Code Debugging And Scrabble Apps With LLMs And AI | STARTUP |
| https://github.com/masto/LED-Marquee | Documenting Personal Project on YouTube and GitHub After Leaving Google | PERSONAL\_PROJECT |
| https://github.com/brendanv/lynxAnd https://github.com/brendanv/lynx-v2 | Personal Read It Later Service Turned SPA for Learning Go | PERSONAL\_PROJECT |
| https://github.com/kilroyjones/series\_game\_from\_scratch | Learning IoUring for Fun with Rust Web Game Development | PERSONAL\_PROJECT |
| https://github.com/hrkck/MyApps/wiki | MyApps App of Apps with Infinite 2D Space and P2P Sync | PERSONAL\_PROJECT |
| https://github.com/rumca-js/Django-link-archive | Title: Developing an RSS Reader and Web Scraper | PERSONAL\_PROJECT |
| https://github.com/DDoS/Cadre | Exploring Enhanced E Ink Displays For Smart Picture Frames | PERSONAL\_PROJECT |
| https://github.com/Vija02/TheOpenPresenter | Open Source Presenter Software for Events and Digital Signage | PERSONAL\_PROJECT |
| https://github.com/itsOwen/CyberScraper-2077 | CyberScraper 2077 Web Scraper Powered By LLM | PERSONAL\_PROJECT |
| https://github.com/fujiapple852/trippy/issues/860 | Forward and Backward Packet Loss Heuristics in Trippy | PERSONAL\_PROJECT |
| https://github.com/bytechefhq/bytechef | ByteChef Open Source API Integration And Workflow Automation Platform | STARTUP |
| https://github.com/AvitalTamir/cyphernetes/ | Cyphernetes Innovative Query Language For Kubernetes API | PERSONAL\_PROJECT |
| https://github.com/willswire/checkd | Exploring Device Authentication with Checkd and Apple's DeviceCheck API | PERSONAL\_PROJECT |
| https://github.com/sdedovic/wgsltoy | WGSL Toy A WebGPU Playground for Shader Development | PERSONAL\_PROJECT |
| https://github.com/Trint-ai/TrintAI | TrintAI Open Source Speech to Text and Analysis Tool | STARTUP |
| https://github.com/trynova/nova | Nova Data-Oriented JavaScript Engine | PERSONAL\_PROJECT |
| https://github.com/AmberSahdev/Open-Interface | Title: Open Source LLM Based Autopilot for Multiple OS | PERSONAL\_PROJECT |
| https://github.com/dvasanth/kadugu | Title: Building a Blazing Speed VPN in Minimal Lines | PERSONAL\_PROJECT |
| https://github.com/ptah-sh/ptah-server | Open Source Alternative To Heroku With Key Features | STARTUP |
| http://github.com/CWood-sdf/banana | HTML Renderer for Neovim Plugins Called Banana | PERSONAL\_PROJECT |
| http://github.com/leftmove/facebook.js | Developing Facebook.js A Modern API Wrapper for Facebook | PERSONAL\_PROJECT |
| https://github.com/amoffat/manifest | Title: Python Library for Simplifying LLM Calls | PERSONAL\_PROJECT |
| https://github.com/humishum/hacker\_news\_keys | Creating a Hacker News Browser Extension Using LLMs | PERSONAL\_PROJECT |
| https://github.com/csjh/pest | Efficient Row-Based Serialization Format with TypeScript Type Safety | PERSONAL\_PROJECT |
| https://github.com/WillAdams/gcodepreviewbig | Creating a Python Enhanced OpenSCAD Library for CNC Projects | PERSONAL\_PROJECT |
| https://github.com/james-a-rob/KodaStream | Title: Interactive Video API for Shoppable Live Streams | STARTUP |
| https://github.com/chaosharmonic/escapeHatch | Developing a Lightweight Job Search Tool With Custom Features | PERSONAL\_PROJECT |
| https://github.com/JUSTSUJAY/Django\_Projects | Embracing Virtual Presence and Mastering Django for Impactful Development | SELF\_IMPROVEMENT |
| https://github.com/memfreeme/memfree | MemFree AI Search Engine for Instant Accurate Answers | STARTUP |
| https://dot-and-box.github.io/dot-and-box/ | Dot And Box Offers Animations Visualizing Algorithms | PERSONAL\_PROJECT |
| https://github.com/mliezun/caddy-snake | Title: Integrating HTTP Requests for Python Apps Using Go | PERSONAL\_PROJECT |
| https://github.com/BigJk/end\_of\_eden | Terminal-Based Deck Builder Game With Mouse Support And Image | PERSONAL\_PROJECT |
| https://github.com/learnbyexample/TUI-apps | Updated Vim Guide Published and New Python TUI App Development | PERSONAL\_PROJECT |
| https://github.com/rybarix/snaptail | Exploring Single Source File Applications | PERSONAL\_PROJECT |
| https://github.com/dvasanth/kadugu | Building Blazing Speed VPN In Less Than 1000 Lines Of Code | PERSONAL\_PROJECT |
| https://github.com/claceio/clace | Clace App Server For Multi Language Containerized Applications | STARTUP |
| https://github.com/leondz/garak | Contribution to LLM Vulnerability Scanner Alongside Final Year Studies | PERSONAL\_PROJECT |
| https://github.com/coreyp1/CTang | Developing CTang A Modern Scripting Language | PERSONAL\_PROJECT |
| https://github.com/ayinke-llc/malak | Title: Developing an OSS Relationship Hub for Founders and Investors | STARTUP |
| https://github.com/fedi-e2ee/public-key-directory-specificat | Building an Open Source Federated Public Key Directory | PERSONAL\_PROJECT |
| https://github.com/laktak/chkbit | Title: Simplifying Cross-Platform Builds by Rewriting chkbit in Go | PERSONAL\_PROJECT |
| https://github.com/latebit/latebit-engine | Game Engine for Coders with Integrated Tools in VSCode | PERSONAL\_PROJECT |
| https://github.com/andrew-johnson-4/lambda-mountain | Verifiable Correctness in Compiler Agnostic Programs | PERSONAL\_PROJECT |
| https://github.com/spirobel/mininext | Mininext Merges Index PHP Simplicity With NPM And TypeScript | PERSONAL\_PROJECT |
| https://github.com/styluslabs/maps | Title: Open Source Maps Application | OTHER |
| https://github.com/thebigG/GunnerIt | Title: Passion Project Scrolling Shooter Inspired By Strike Gunner STG | PERSONAL\_PROJECT |
| https://github.com/itissid/privyloci | Zero Trust Proposal for Mobile Location Permission Control | PERSONAL\_PROJECT |
| https://github.com/brainless/dwata/tree/feature/prepare\_mvp\_ | Open Source App for Emails with AI Features | STARTUP |
| https://github.com/rprtr258/pm | Title: Simple Linux Process Manager | PERSONAL\_PROJECT |
| https://github.com/jasiek/webprog-anytone | Programming Anytone AT878 DMR Radios via Web Browser | PERSONAL\_PROJECT |
| https://github.com/moj-analytical-services/splink | Version 4 Released for Data Deduplication and Linkage Library | PERSONAL\_PROJECT |
| https://github.com/preludejs | Developing Standard Libraries for TypeScript and JavaScript | PERSONAL\_PROJECT |
| https://github.com/b00bl1k/uwan | LoRaWAN Node Device Library Development and Documentation Summary | PERSONAL\_PROJECT |
| https://github.com/codetiger/PowerTiger | PowerTiger Open Source Energy Monitoring Solution Using RPi Pico W | PERSONAL\_PROJECT |
| https://github.com/laudspeaker/laudspeaker | Techno Thriller Panopticon Explores Encryption and Espionage | PERSONAL\_PROJECT |
| https://github.com/certeu/morio | Morio Streamlines Observability Data for Traditional On-Premises Infrastructure | PERSONAL\_PROJECT |
| https://github.com/iterative/datachain | Title: Out Of Memory Dataframe For Wrangling Unstructured Data At Scale | OTHER |
| https://github.com/petabyt/libui-touch | Unfinished C-Based Alternative to React Native | PERSONAL\_PROJECT |
| https://github.com/David-OConnor/plascad | PlasCAD Molecular Biology Plasmid Editor Seeking Feedback | PERSONAL\_PROJECT |
| https://github.com/tttapa/Control-Surface | Repurposing a Teensy Synth into a MIDI Controller | PERSONAL\_PROJECT |
| https://github.com/cmakafui/batchwizard | BatchWizard A Command Line Tool for OpenAI Batch Jobs | PERSONAL\_PROJECT |
| https://github.com/ruuda/rcl | Title: Enhancements to RCL Language with Float Support and Query Shorthand | PERSONAL\_PROJECT |
| https://github.com/Eccentric-Anomalies/Tungsten-Moon-Demo-Re | Tungsten Moon VR Desktop Spaceflight Simulator Nears Early Access Release | PERSONAL\_PROJECT |
| https://github.com/matry/editor | Keyboard-Driven UI Editor Inspired by Vim and Webflow | PERSONAL\_PROJECT |
| https://github.com/ssherman/weighted\_list\_rank | Title: Improving Book Ranking Algorithm Through Collaboration With Data Scientists | PERSONAL\_PROJECT |
| https://github.com/DefGuard | DefGuard Open Source SSO Integrating Wireguard and OIDC | STARTUP |
| https://github.com/glaretechnologies/substrataCustom | Open Source Metaverse With Custom 3D Engine And Voice Chat | OTHER |
| https://github.com/ibizaman/selfhostblocks | Self Host Blocks A Modular Server Management Tool | PERSONAL\_PROJECT |
| https://github.com/golang-malawi/qatarina | Building a User Acceptance Testing Platform with Go and Encouraging Go Adoption | PERSONAL\_PROJECT |
| https://github.com/curveball/a12n-server | Title: Open Source OAuth2 Server Competing with Auth0 | OTHER |
| https://github.com/jaronilan/stories | Title: Finishing a Yearlong Short Story About SEO | PERSONAL\_PROJECT |
| https://github.com/featurevisor/featurevisor | Title: Open Source Tool for Declarative Feature Management | STARTUP |
| https://github.com/patrulek/modernRX | Upgrading AVX2 to AVX512 in RandomX Algorithm Reimplementation | PERSONAL\_PROJECT |
| https://github.com/mseravalli/grizol | Grizol: Syncthing Compatible Client Leveraging Rclone Backends | PERSONAL\_PROJECT |
| https://github.com/beef331/potato | Hot Code Reloading Library for Nim Game Framework | PERSONAL\_PROJECT |
| https://github.com/chrisdavies/atomic-css | Zero-Dependency Bun Application with Tailwind-Like Layer | PERSONAL\_PROJECT |
| https://github.com/carlnewton/habitat | Developing Habitat A Self Hosted Social Platform | PERSONAL\_PROJECT |
| https://github.com/achristmascarl/rainfrog | Rainfrog Postgres Management Terminal with Vim-like Keybindings | PERSONAL\_PROJECT |
| https://github.com/aravinda0/qtile-bonsai | Qtile Bonsai Completion and Future PDF Parser Plans | PERSONAL\_PROJECT |
| https://github.com/emlearn/emlearn-micropython | MicroPython for Machine Learning on Microcontroller Sensors | PERSONAL\_PROJECT |
| https://github.com/julien040/anyquery | Building Anyquery An SQL Query Engine For Diverse Data Sources | PERSONAL\_PROJECT |
| https://github.com/elijah-potter/harper | Improving On-Device Grammar Checker with Minimal Resource Use | PERSONAL\_PROJECT |
| https://bgammon.org/code | Open Source Online Backgammon Project Inspired By Lichess | PERSONAL\_PROJECT |
| https://gist.github.com/skittleson/705624a8f6967187096091cbd | Bluetooth Low Energy Wall of Sheep Toy App | PERSONAL\_PROJECT |
| https://github.com/elixir-error-tracker/error-tracker | Elixir Based Error Reporting Solution | PERSONAL\_PROJECT |
| http://github.com/kviklet/kviklet | Streamlining SQL Query Reviews to Prevent Costly Errors | PERSONAL\_PROJECT |
| https://github.com/opslane/opslane | Building a Copilot for Oncall Engineers Reducing Grunt Work | STARTUP |
| https://github.com/pierreyoda/hncli | Developing a Rust Based Hacker News TUI Reader | PERSONAL\_PROJECT |
| https://github.com/ibudiallo/automated-agents-book | Creating A Comprehensive Guide On Building Effective Chatbots | PERSONAL\_PROJECT |
| https://github.com/jhspetersson/git-task | Git Task: Local Task Manager and Bug Tracker in Git | PERSONAL\_PROJECT |
| https://github.com/madprops/goldie | Firefox Vertical Tabs Powerful Python Chat Client Nim Text Finder | PERSONAL\_PROJECT |
| https://github.com/av/harbor | Building a Toolkit to Save Time Using Local LLMs | PERSONAL\_PROJECT |
| https://github.com/captn3m0/ideas | Curated Events Platform For Bangalore Using Open Source Tools | STARTUP |



