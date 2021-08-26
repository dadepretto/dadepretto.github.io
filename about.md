---
layout: page
title: "About me"
permalink: /about/
---

Hi there!

I'm Davide De Pretto, a data engineer and backend software developer from Verona, Italy.

I know and work primarily with T-SQL/SQL Server, Typescript/NodeJS, and the Azure ecosystem. Still, I'm also highly interested in C#/.NET for its expressiveness and power tooling and the rest of Microsoft Data Platform, such as CosmosDB and other storage solutions.

Currently, I'm working in the R&D department at [Trueblue](https://truebluecorp.com), architecting and building sizeable enterprise-grade data platforms and web services for the pharmaceutical industry. I'm also studying to get some Microsoft Azure certifications so that I can flex them on Linkedin.

When I'm not sitting at my desk building stuff, I enjoy listening to music, eating fine food, and drinking good wine.

## Work experiences

### (2016 - 2019) Teacher & Full Stack Developer @ SI-FA

I started working in this small local business as a part-time computer science teacher right after my graduation. At the time, I had studied to become an audio engineer, so I was trying to leverage my computer science knowledge while looking around for career opportunities in my field of choice.

Six months in, and I was still in the same situation. At that point, I wanted at least a higher income, so I proposed to the business owner to help them with their digital transformation by moving from their old spreadsheets and paperwork to an integrated internally-developed CRM.

So I started developing this web-based software with MySQL, PHP, and JQuery to track students, parents, teachers, lessons, payments, and paychecks. After the initial go-live, I spent the next couple of years between my teacher role and my new technical role, developing new features for this CRM and helping the business out with all sorts of technical challenges. Those also included deploying a Microsoft 365 tenant for about 50 people and migrating their documents to OneDrive and SharePoint for collaboration.

All and all, although that CRM is for sure not one of the most complex out there, counting less than a dozen tables in the database, I'm pretty proud of what I have done and thankful for the experience. In fact, despite being developed entirely on my own using only online learning resources and not being actively developed since I left, it's still in use today, and it serves its purpose with dignity.

### (2019 - today) Backend & Database Developer @ Trueblue

My career in Trueblue started as a full-stack developer inside the legacy CRM team, working on a NodeJS backend with lots of T-SQL and a custom web front end platform.

After a year, I got promoted to the R&D department, where I'm still working. Here I got the opportunity to develop some of the new flagship company products, including AiDEA. Specifically, I'm co-architecting and developing their backend infrastructures, from data ingestion using Azure Data Factory to databases to the API. Of course, the project is ambitious and has many moving parts, being a multi-tenant product aimed at big corporations.

For this reason, during this experience, I've had the opportunity to learn a lot, including some highlights that I'd like to share.

First, I'm proud of the architecture of the APIs: I used a strict clean architecture to create a multi-tenant backend where each tenant can connect to a different enterprise data source (Microsoft Dynamics, Salesforce, etc...), exposing a standard, shared interface to the mobile app to perform CRUD operations against, regardless of the source data system. I'm particularly pleased with how its layered, domain-driven structure helps us keep new developments neat and how it helps new developers to onboard with ease.

Then, I'm thankful for all the database knowledge I've got doing this project, mainly regarding T-SQL and Azure SQL Database. From database versioning using Database Projects and dacpacs to advanced database programming, I feel like I can interact with databases with great confidence now. I also developed a set of internal utility procedures to automate all kinds of recurring tasks, with a fascinating excursus on SQL Server's extended events, which are as helpful as complex and sadly under-used.

Lastly, I enjoyed creating all the GitHub Actions to automatically build and deploy all the backend components, including Docker containers, databases, and even Azure Data Factories in a CI/CD fashion. This last one is the most modern-feeling achievement I got, and seeing everything move and deploy automatically after a pull request is for sure one of the best and most exciting feelings I had, at least work-wise.
