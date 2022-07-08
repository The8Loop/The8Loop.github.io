---
title: "SQL Stored Procedures"
author: E.S. Luper
date: 2022-07-07T14:42:27-05:00
draft: false
---

# SQL-Stored-Procedures
The back-end for the Council of Yggdrasil website was written using .Net framework. When one uses such a framework to set up end points and communicate to a server, one likely uses Object-Relational Mapping (ORM). The .Net ORM used for Council of Yggdrasil is Entity Framework Core. An ORM allows one to make queries to a database using an object-oriented language versus, for example, SQL. While ORM comes with the advatange of writing in an OOP language and keeping query logic saved on the back-end level, complex queries are more efficiently done in the database side using stored procedures, an executable routine written in the database language (SQL). This also has the advantage of maintainability. If the ORM or back-end framework are changed, the complex procedures are safe from needing to be rewritten or transfered, as SQL isn't going anywhere. SQL's universality also keeps it independent from a developer's choice or understanding of a specific OOP language, making it more accessible.

An example of the use of a stored-procedure for Council of Yggdrasil is in a leaderboard of top contributors. The database includes a User, and a Money table, and may contain the following entries:

User:

| Id | Name            | IsActive |
|----|-----------------|----------|
| 1  | Thorak Icestorm | 1        |
| 2  | Sonata Icestorm | 1        |
| 3  | Zia Mordrem     | 1        |
| 4  | Kyler Blint     | 1        |

Money:

| Id | Contribution | Date       | ContributionTypeId | UserId |
|----|--------------|------------|--------------------|--------|
| 1  | 250          | 2022-07-07 | 1                  | 1      |
| 2  | -100         | 2022-07-07 | 4                  | 1      |
| 3  | -50          | 2022-07-07 | 4                  | 1      |
| 4  | -100         | 2022-07-07 | 4                  | 3      |
| 5  | 150          | 2022-07-07 | 1                  | 2      |
| 6  | -30          | 2022-07-07 | 3                  | 4      |
| 7  | 10           | 2022-07-07 | 2                  | 1      |
| 8  | 10           | 2022-07-07 | 2                  | 1      |

\
A stored procedure can be written to handle the joining of these tables by the user id, considering only personal contributions and withdrawals, and grouping the entries by user id while summing.

``` sql
CREATE DEFINER=`root`@`localhost` PROCEDURE `Leaderboard`()
BEGIN
SELECT 
	SUM(money.Contribution) total,
    users.Name
FROM 
	money
INNER JOIN 
	users 
    ON users.Id = money.UserId
WHERE 
	(ContributionTypeId = 1 
	OR ContributionTypeId = 3)
GROUP BY UserId
HAVING total > 0
LIMIT 3;
END
```

Resulting Table:

| Name            | Total    |
|-----------------|----------|
| Thorak Icestorm | 250      |
| Sonata Icestorm | 150      |

To call the stored procedure in .Net

``` C#
	public IEnumerable<Leaderboard> GetLeaderboard()
    {
      return _context.Set<Leaderboard>().FromSqlRaw($"CALL Leaderboard();").ToList();
    }
```