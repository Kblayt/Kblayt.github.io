---
title: MySQL50题
date: 2023-03-25 13:09:36
categories:
    - 数据库
tags:
    - 数据库
    - MySQL
---

**\#1、查询“001”课程比“002”课程成绩高的所有学生的学号；**

```mysql
select distinct s1.SID
from score s1
where
(select s2.score
from score s2
where s2.CID="001" and s1.SID=s2.SID)
(select s3.score
from score s3
where s3.CID="002" and s1.SID=s3.SID);
```

**\#2、查询平均成绩大于60分的同学的学号和平均成绩；**

```mysql
select sid,avg(score) as a
from score
group by score.sid
having a>60
order by a desc;
```

**\#3、查询所有同学的学号、姓名、选课数、总成绩；**

```mysql
select s.SID,s.Sname,count(1),sum(score)
from student as s
left join score as sc
on s.SID=sc.SID
left join course as c
on sc.CID=c.CID
group by s.SID;
```

**\#4、查询姓“李”的老师的个数；**

```mysql
select count(1)
from teacher as t
where t.Tname like "李%";
```

**\#5、查询没学过“叶平”老师课的同学的学号、姓名；**

```mysql
select student.SID,student.Sname
from student
where student.Sname not in
(
select distinct s.Sname
from student as s
left join score as sc
on s.SID=sc.SID
left join course as c
on sc.CID=c.CID
left join teacher as t
on c.TID=t.TID
where t.Tname='叶平'
);
```

**\#6、查询学过“001”并且也学过编号“002”课程的同学的学号、姓名**

```mysql
select s.SID,s.Sname
from student as s
where s.SID in (
select sc1.SID
from score sc1,score sc2
where sc1.SID=sc2.SID and sc1.CID=001 and sc2.CID=002
);
```

**\#7、查询学过“叶平”老师所教的所有课的同学的学号、姓名；**

```mysql
SELECT sid,sname FROM Student s WHERE NOT EXISTS
(
SELECT cid
FROM Course c
INNER JOIN Teacher t
ON c.TID=t.TID
WHERE t.TName='叶平' AND CID NOT IN
(SELECT CID FROM score WHERE SID=s.SID)
);
```

**\#8、查询课程编号“002”的成绩比课程编号“001”课程低的所有同学的学号、姓名；**

```mysql
select DISTINCT s.SID,s.Sname
from score s1
left join student s
on s.SID=s1.SID
where
(select s2.score
from score s2
where s2.CID="001" and s1.SID=s2.SID)
(select s3.score
from score s3
where s3.CID="002" and s1.SID=s3.SID);
```

**\#9、查询所有课程成绩小于60分的同学的学号、姓名；**

```mysql
select a.SID,a.Sname
from student as a
where a.SID not in(
select distinct s.SID
from student as s
left join score as sc
on s.SID=sc.SID
where sc.score>=60
order by s.SID asc
)
order by a.SID asc;
```

**\#10、查询没有学全所有课的同学的学号、姓名；**

```mysql
select s.SID,s.Sname
from student as s
where
(select count() from score as sc where s.SID=sc.SID)<(select count() from course);
```

**\#11、查询至少有一门课与学号为“1001”的同学所学相同的同学的学号和姓名；**

```mysql
select distinct a.SID,a.Sname
from student as a
left join score as b
on a.SID=b.SID
where b.CID in(
select score.CID
from score
where score.SID=1001
);
```

**\#12、查询没有学过学号为“1001”同学所有门课的其他同学学号和姓名；**

```mysql
select distinct s.SID,s.Sname
from student as s
left join score as sc1
on sc1.SID=s.SID
where s.SID not in (
select  sc.SID
from score as sc
where sc.CID in (
select CID from score where SID=1001
)
group by sc.SID
);
```

**\#13、把“SC”表中“叶平”老师教的课的成绩都更改为此课程的平均成绩；**

```mysql
update score s1
inner join(
select CID,avg(score) as ascore from score where CID in
(
select c.CID
from course as c,teacher as t
where c.TID=t.TID and t.Tname='叶平'
)
group by CID
) as s2
set s1.score=s2.ascore where s1.CID=s2.CID;
```

**\#14、查询和“1002”号的同学学习的课程完全相同的其他同学学号和姓名；**

```mysql
select s.SID '学号',s.Sname '姓名'
from student as s ,score as sc1,
(select SID as si,COUNT(1) as co
from score
group by SID
)as a
where  s.SID=sc1.SID
and a.si=sc1.SID
and  sc1.cid in
(
select sc.CID
from score as sc
where sc.sID=1001
)
and a.co= (select COUNT(1) from score where SID=1001)
group by s.SID
having COUNT(1)=(select COUNT(1) from score where SID=1001);
```

**\#15、删除学习“叶平”老师课的SC表记录;**

```mysql
delete from score as sc
where sc.CID in(
select c.CID
from course as c
left join teacher as t
on c.TID=t.TID
where t.Tname='叶平'
);
```

**\#16、向SC表中插入一些记录，这些记录要求符合以下条件：没有上过编号“003”课程的同学学号的2号课的平均成绩；**

```mysql
insert score
select distinct sc2.SID,2,ascore.a
from score as sc2, (
select avg(sc.score) as a
from score as sc
where sc.CID=002
)  as ascore
where sc2.SID not  in(
(select sc.SID
from score as sc
where sc.CID=003
)
)
group by sc2.SID
```

**\#17、按平均成绩从高到低显示所有学生的“数据库”、“企业管理”、“英语”三门的课程成绩，按如下形式显示： 学生ID,数据库,企业管理,英语,有效课程数,有效平均分**

//?????????????????????????????

```mysql
SELECT distinct sid '学生ID',
(SELECT score FROM score a,course b WHERE a.cid=b.cid AND b.cname='数据库' AND a.sid=c.sid)'数据库',
(SELECT score FROM score a,course b WHERE a.cid=b.cid AND b.cname='企业管理' AND a.sid=c.sid)'企业管理',
(SELECT score FROM score a,course b WHERE a.cid=b.cid AND b.cname='英语' AND a.sid=c.sid)'英语',
(SELECT COUNT(1) FROM score a,course b WHERE a.cid=b.cid AND (b.cname='数据库' OR b.cname='企业管理'OR b.cname='英语'  )AND a.sid=c.sid)'有效课程数',
(SELECT AVG(score) FROM score a,course b WHERE a.cid=b.cid AND (b.cname='数据库' OR b.cname='企业管理'OR b.cname='英语'  ) AND a.sid=c.sid) 有效平均分
FROM score c
ORDER BY 有效平均分 DESC;
```

**\#18、查询各科成绩最高和最低的分：以如下形式显示：课程ID，最高分，最低分**

```mysql
select sc.CID,max(sc.score),min(sc.score)
from score as sc
group by sc.CID
order by sc.CID asc;
```

**\#19、按各科平均成绩从低到高和及格率的百分数从高到低顺序**

\#case when then else

```mysql
select sc.CID,avg(sc.score) as '平均成绩',100**sum(case when sc.score>=60 then 1 else 0 end)/count(1) as 及格率from score as scgroup by sc.CIDorder by avg(sc.score) asc,及格率 desc;#20、查询如下课程平均成绩和及格率的百分数(用"1行"显示): 企业管理（001），马克思（002），OO&UML （003），数据库（004）selectsum(case when sc.CID=001 then sc.score else 0 end)/sum(case when sc.CID=001 then 1 else 0 end) as '企业管理平均成绩',100**sum(case when sc.CID=001 and sc.score>=60 then 1 else 0 end)/sum(case when sc.CID=001 then 1 else 0 end)as '企业管理及格率',
sum(case when sc.CID=002 then sc.score else 0 end)/sum(case when sc.CID=002 then 1 else 0 end) as '马克思平均成绩',
100**sum(case when sc.CID=002 and sc.score>=60 then 1 else 0 end)/sum(case when sc.CID=002 then 1 else 0 end)as '马克思及格率',sum(case when sc.CID=003 then sc.score else 0 end)/sum(case when sc.CID=003 then 1 else 0 end) as 'UML平均成绩',100**sum(case when sc.CID=003 and sc.score>=60 then 1 else 0 end)/sum(case when sc.CID=003 then 1 else 0 end)as 'UML及格率',
sum(case when sc.CID=004 then sc.score else 0 end)/sum(case when sc.CID=004 then 1 else 0 end) as '数据库平均成绩',
100*sum(case when sc.CID=004 and sc.score>=60 then 1 else 0 end)/sum(case when sc.CID=004 then 1 else 0 end)as '数据库及格率'
from score as sc;
```

**/*21、查询不同老师所教不同课程平均分从高到低显示**

```mysql
SELECT max(Z.TID) AS 教师ID,MAX(Z.Tname) AS 教师姓名,C.CID AS 课程ＩＤ,MAX(C.Cname) AS 课程名称,AVG(Score) AS 平均成绩 */
SELECT t.TID AS '教师ID',t.Tname AS '教师姓名',c.CID AS '课程ID',C.Cname AS '课程名称',avg(sc.score) AS '平均成绩'
from score as sc
left join course as c
on sc.CID=c.CID
left join teacher as t
on c.TID=t.TID
group by c.TID,c.CID
order by avg(sc.score) desc;
```

**\#22、查询如下课程成绩第 3 名到第 6 名的学生成绩单：企业管理（001），马克思（002），UML （003），数据库（004）**

```mysql
SELECT DISTINCT s.cid,
(SELECT s1.score FROM score s1 WHERE s1.cid = s.cid ORDER BY s1.score DESC LIMIT 2,1)'3',
(SELECT s1.score FROM score s1 WHERE s1.cid = s.cid ORDER BY s1.score DESC LIMIT 3,1)'4',
(SELECT s1.score FROM score s1 WHERE s1.cid = s.cid ORDER BY s1.score DESC LIMIT 4,1)'5',
(SELECT s1.score FROM score s1 WHERE s1.cid = s.cid ORDER BY s1.score DESC LIMIT 5,1)'6'
FROM score s
WHERE s.cid IN (1,2,3,4)
order by s.cid
select t1.SID,
(select sname from student as stu where stu.SID=t1.SID) as 姓名,
(select Cname from Course as co where co.CID=t1.CID) as 课程名,
t1.score  as 成绩
from score t1
where (select count(1) from score where CID=t1.CID and score>=t1.score) between 2 and 5
and CID in ('1','2','3','4')
order by CID,t1.score desc
```

**\#23、统计列印各科成绩,各分数段人数:课程ID,课程名称,[100-85],[85-70],[70-60],[ <60]**

**\#case when between 小数 and 大数 then 1 else 0 end**

```mysql
select
sc.CID as '课程ID',
c.Cname as '课程名称',
sum(case when sc.score BETWEEN 85 and 100 then 1 else 0 end) as '[100-85]',
sum(case when sc.score BETWEEN 70 and 85 then 1 else 0 end) as '[85-70]',
sum(case when sc.score BETWEEN 60 and 70 then 1 else 0 end) as '[70-60]',
sum(case when sc.score BETWEEN 0 and 60 then 1 else 0 end) as '[60-0]'
from score as sc,course as c
where sc.CID=c.CID
group by sc.CID
order by sc.CID asc;
```

**\#24、查询学生平均成绩及其名次**

```mysql
set @n=0;
select a.SID,a.ascore,(@n:=@n+1) as 排名
from(
select SID,avg(score) as ascore
from score
group by SID
order by ascore desc
)as a
```

**\#25、查询各科成绩前三名的记录:(不考虑成绩并列情况)**

//??????????????????????????????????????????

```mysql
select CID,score
from score as sc
where (select count(1) from score as sc1 where sc.CID=sc1.CID and sc.score
order by CID,score;
```

**\#26、查询每门课程被选修的学生数**

```mysql
select sc.CID,c.Cname,sum(case when sc.score then 1 else 0 end) as 选修的学生数
from score as sc,course as c
where sc.CID=c.CID
group by sc.CID
order by sc.CID
```

**\#27、查询出只选修了一门课程的全部学生的学号和姓名**

```mysql
select s.SID,s.Sname,sum(case when sc.CID then 1 else 0 end) as 每个人选修的课程数
from student as s,score as sc
where s.SID=sc.SID
group by s.SID
having 每个人选修的课程数=1;
```

**\#28、查询男生、女生人数**

```mysql
select sum(case when s.Ssex='男' then 1 else 0 end) as 男生人数,sum(case when s.Ssex='女' then 1 else 0 end) as 女生人数
from student as s
```

**\#29、查询姓“张”的学生名单**

```mysql
select *
from student as s
where s.Sname like '张%';
```

**\#30、查询同名同性学生名单，并统计同名人数**

```mysql
select *,count(1)
from student as s1,student as s2
where s1.SID!=s2.SID and s1.Sname=s2.Sname
insert student values (9999,'张无忌',22,'男');
```

**\#31、1981年出生的学生名单(注：Student表中Sage列的类型是datetime)**

\#运行时将student.Sage类型改为datetime类型

```mysql
select *
from student as s
where YEAR(s.Sage)=1981;
```

**\#32、查询每门课程的平均成绩，结果按平均成绩升序排列，平均成绩相同时，按课程号降序列**

```mysql
select sc.CID as 课程号,avg(sc.score) as 平均成绩
from score as sc
group by sc.CID
order by 平均成绩 desc,课程号 asc;
```

**\#33、查询平均成绩大于85的所有学生的学号、姓名和平均成绩**

```
select s.SID,s.Sname,avg(sc.score) as 个人平均成绩
from student as s
left join score as sc
on s.SID=sc.SID
group by sc.SID
having 个人平均成绩>85
insert score values(9999,1,95),(9999,2,85);
insert student values(9999,'刘红舸',22,'男');
```

**\#34、查询课程名称为“数据库”，且分数低于60的学生姓名和分数**

```mysql
select s.Sname,sc.score
from student as s
left join score as sc
on s.SID=sc.SID
left join course as c
on sc.CID=c.CID
where c.Cname='数据库' and sc.score<60;
```

**\#35、查询所有学生的选课情况；**

```mysql
select distinct s.Sname,sc.CID,c.Cname
from student as s
left join score as sc
on s.SID=sc.SID
left join course as c
on sc.CID=c.CID
order by c.CID asc;
```

**\#36、查询任何一门课程成绩在70分以上的姓名、课程名称和分数；**

```mysql
SELECT s.SID, s.Sname,c.cname,sc1.score
FROM score AS sc1, course c ,student s
WHERE sc1.sid=s.SID
AND c.cid=sc1.cid
AND s.SID NOT in
(
SELECT sc.SID
FROM score as sc
WHERE sc.score<70
);
```

**\#37、查询不及格的课程，并按课程号从大到小排列**

```mysql
select s.Sname,Cname,sc.score
from student as s
left join score as sc
on s.SID=sc.SID
left join course as c
on sc.CID=c.CID
where sc.score<60
order by sc.CID desc;
```

**\#38、查询课程编号为003且课程成绩在80分以上的学生的学号和姓名；**

```mysql
select s.Sname,sc.CID,s.SID,sc.score
from student as s
left join score as sc
on s.SID=sc.SID
where sc.CID=003 and sc.score>80;
```

**\#39、求选了课程的学生人数**

```mysql
select count(1) 选了课的学生人数
from(
select 1
from score
group by SID) as a;
```

**\#40、查询选修“叶平”老师所授课程的学生中，成绩最高的学生姓名及其成绩**

```mysql
SELECT *
FROM score as sc
LEFT JOIN student s ON sc.sid = s.sid
LEFT JOIN course c ON sc.cid = c.cid
LEFT JOIN teacher t ON c.tid = t.tid
WHERE t.tname = '叶平'
AND sc.score = (SELECT MAX(score) FROM score s1 WHERE sc.cid = s1.cid)
```

**\#41、查询各个课程及相应的选修人数**

```mysql
select c.Cname,sum(case when sc.CID then 1 else 0 end) as 选修人数
from score as sc
left join course as c
on sc.CID=c.CID
group by sc.CID;
```

**\#42、查询不同课程成绩相同的学生的学号、课程号、学生成绩**

```mysql
select distinct sc.SID,sc.CID,sc.score
from score as sc,score as sc1
where sc.CID!=sc1.CID and sc.score=sc1.score
order by sc.score desc;
```

**\#43、查询每门功成绩最好的前两名**

//??????????????????????????????????????????

```mysql
select CID,score
from score as sc
where (select count(1) from score as sc1 where sc.CID=sc1.CID and sc.score
order by CID,score;
```

**\#44、统计每门课程的学生选修人数（超过10人的课程才统计）。要求输出课程号和选修人数，查询结果按人数降序排列，若人数相同，按课程号升序排列**

```mysql
select sc.CID,sum(case when sc.CID then 1 else 0 end) as 选修人数
from score as sc
inner join course as c
on sc.CID=c.CID
group by c.CID
having 选修人数>=5
order by 选修人数 desc,c.CID asc;
```

**\#45、检索至少选修两门课程的学生学号**

```mysql
select sc.SID,count(1)
from score as sc
group by sc.SID
having count(1)>=2
```

**\#46、查询全部学生都选修的课程的课程号和课程名**

```mysql
SELECT a.cid,a.cname FROM course a
WHERE  NOT EXISTS
(
SELECT *
FROM student
WHERE sid not  IN(
SELECT sid
FROM score
WHERE cid=a.cid
)
);
SELECT a.课程号,COUNT(1) AS '被选的人数'
FROM(
SELECT s.SID SID,s.Sname Sname,sc.CID 课程号,COUNT(sc.CID)
FROM student s
LEFT JOIN score as sc
ON s.SID = sc.SID
GROUP BY s.SID,sc.CID
ORDER BY s.SID
)AS a
GROUP BY a.课程号
HAVING COUNT(1)=(SELECT COUNT(1) FROM student)
insert score VALUES (1001,1,80)
```

**#47、查询没学过“叶平”老师讲授的任一门课程的学生姓名**

```
select s.SID,s.Sname
from student as s
where s.SID not in (
select sc.SID
from score as sc
left join course as c
on sc.CID=c.CID
left join teacher as t
on c.TID=t.TID
where t.Tname='叶平'
);
```

**\#48、查询两门以上不及格课程的同学的学号及其平均成绩**

```mysql
select sc.SID,avg(sc.score),sum(case when sc.score<=60 then 1 else 0 end) as 不及格门数
from score as sc
group by sc.SID
having 不及格门数>=2
```

**\#49、检索“004”课程分数小于60，按分数降序排列的同学学号**

```mysql
select sc.SID,sc.score
from score as sc
where sc.CID=004 and sc.score<60
order by sc.score desc;
```

**\#50、删除“002”同学的“001”课程的成绩**

```mysql
delete from score as sc
where sc.SID=002 and sc.CID=001;
```

