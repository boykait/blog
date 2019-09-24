-- 建表
-- 学生表
CREATE TABLE IF NOT EXISTS `Student` (
	`s_id` VARCHAR(20),
	`s_name` VARCHAR(20) NOT NULL DEFAULT '',
	`s_birth` VARCHAR(20) NOT NULL DEFAULT '',
	`s_sex` VARCHAR(10) NOT NULL DEFAULT '',
	PRIMARY KEY(`s_id`)
);
-- 课程表
CREATE TABLE IF NOT EXISTS `Course`(
	`c_id`  VARCHAR(20),
	`c_name` VARCHAR(20) NOT NULL DEFAULT '',
	`t_id` VARCHAR(20) NOT NULL,
	PRIMARY KEY(`c_id`)
);
-- 教师表
CREATE TABLE IF NOT EXISTS `Teacher`(
	`t_id` VARCHAR(20),
	`t_name` VARCHAR(20) NOT NULL DEFAULT '',
	PRIMARY KEY(`t_id`)
);
-- 成绩表
CREATE TABLE IF NOT EXISTS `Score`(
	`s_id` VARCHAR(20),
	`c_id`  VARCHAR(20),
	`s_score` INT(3),
	PRIMARY KEY(`s_id`,`c_id`)
);
-- 插入学生表测试数据
insert into Student values('01' , '赵雷' , '1990-01-01' , '男');
insert into Student values('02' , '钱电' , '1990-12-21' , '男');
insert into Student values('03' , '孙风' , '1990-05-20' , '男');
insert into Student values('04' , '李云' , '1990-08-06' , '男');
insert into Student values('05' , '周梅' , '1991-12-01' , '女');
insert into Student values('06' , '吴兰' , '1992-03-01' , '女');
insert into Student values('07' , '郑竹' , '1989-07-01' , '女');
insert into Student values('08' , '王菊' , '1990-01-20' , '女');
-- 课程表测试数据
insert into Course values('01' , '语文' , '02');
insert into Course values('02' , '数学' , '01');
insert into Course values('03' , '英语' , '03');

-- 教师表测试数据
insert into Teacher values('01' , '张三');
insert into Teacher values('02' , '李四');
insert into Teacher values('03' , '王五');

-- 成绩表测试数据
insert into Score values('01' , '01' , 80);
insert into Score values('01' , '02' , 90);
insert into Score values('01' , '03' , 99);
insert into Score values('02' , '01' , 70);
insert into Score values('02' , '02' , 60);
insert into Score values('02' , '03' , 80);
insert into Score values('03' , '01' , 80);
insert into Score values('03' , '01' , 80);
insert into Score values('03' , '02' , 80);
insert into Score values('03' , '03' , 80);
insert into Score values('04' , '01' , 50);
insert into Score values('04' , '02' , 30);
insert into Score values('04' , '03' , 20);
insert into Score values('05' , '01' , 76);
insert into Score values('05' , '02' , 87);
insert into Score values('06' , '01' , 31);
insert into Score values('06' , '03' , 34);
insert into Score values('07' , '02' , 89);
insert into Score values('07' , '03' , 98);



                                                             -- 1、查询"01"课程比"02"课程成绩高的学生的信息及课程分数
-- SELECT stu.*, newScore.s_score FROM Student AS stu, 
-- (SELECT s1.s_id, s1.s_score FROM 
-- (SELECT s_id, s_score FROM Score WHERE c_id='01') AS s1 ,
-- (SELECT s_id, s_score FROM Score WHERE c_id='02') AS s2 
-- WHERE  s1.s_id = s2.s_id and  s1.s_score > s2.s_score
--  ) AS newScore WHERE stu.s_id = newScore.s_id

-- 2、查询平均成绩大于等于60分的同学的学生编号和学生姓名和平均成绩
-- SELECT  stu.s_id, stu.s_name, s.score  from
-- (SELECT AVG(s_score) AS score, s_id FROM Score GROUP BY s_id HAVING AVG(s_score) >= 60) as s, Student AS stu
-- WHERE s.s_id = stu.s_id ;


-- 3、查询平均成绩小于60分的同学的学生编号和学生姓名和平均成绩 包括有成绩的和无成绩的)


-- 4、查询所有同学的学生编号、学生姓名、选课总数、所有课程的总成绩
-- SELECT stu.s_id AS 编号, stu.s_name AS 姓名, COUNT(s.c_id) AS 选课总数, SUM(s.s_score) AS 总成绩 FROM Student AS stu, Score AS s WHERE stu.s_id = s.s_id GROUP BY s.s_id

-- 5、查询姓李老师的数量
-- SELECT COUNT(*) AS 李姓老师数量 FROM Teacher WHERE t_name like '李%';

-- 6、查询学过"张三"老师授课的同学的信息
-- SELECT stu.s_id, stu.s_name, stu.s_brith, stu.s_sex FROM Student AS stu, 
--   (SELECT s_id FROM Score AS s,  
--       (SELECT c_id FROM Course WHERE t_id in (SELECT t_id FROM Teacher WHERE t_name='张三' )) AS c
--     WHERE s.c_id = c.c_id) AS ids
-- WHERE stu.s_id = ids.s_id;

-- 7、查询没学过"张三"老师授课的同学的信息
--  SELECT DISTINCT stu.s_id, stu.s_name, stu.s_brith, stu.s_sex FROM Student AS stu, 
--     (SELECT s_id FROM Score AS s,  
--       (SELECT c_id FROM Course WHERE t_id in (SELECT t_id FROM Teacher WHERE t_name != '张三' )) AS c
--     WHERE s.c_id = c.c_id) AS ids
--  WHERE stu.s_id = ids.s_id;

-- 8、查询学过编号为"01"并且也学过编号为"02"的课程的同学的信息
-- SELECT stu.* FROM Student AS stu, 
--  (SELECT s1.s_id FROM Score AS s1, Score as s2 WHERE s1.s_id = s2.s_id AND s1.c_id='01' AND s2.c_id='02') AS ids
-- WHERE stu.s_id = ids.s_id;
-- IN 是否适合大数据量的查询操作
-- SELECT stu.* FROM Student AS stu WHERE stu.s_id IN (SELECT s1.s_id FROM Score AS s1, Score as s2 WHERE s1.s_id = s2.s_id AND s1.c_id='01' AND s2.c_id='02')


-- 9、查询学过编号为"01"但是没有学过编号为"02"的课程的同学的信息



-- 10、查询没有学全所有课程的同学的信息
-- -- SELECT stu.*, a.c_count FROM Student AS stu, 
-- --    (SELECT s.s_id, COUNT(*) AS c_count FROM Score AS s GROUP BY s.s_id HAVING c_count < (SELECT COUNT(c_id) FROM Course)) AS a
-- -- WHERE stu.s_id = a.s_id;

-- 11.查询至少有一门课与学号为“01”的同学所学相同的同学的学号和姓名；
-- SELECT DISTINCT stu.s_id, stu.s_name FROM Student AS stu, Score AS s WHERE stu.s_id=s.s_id AND s.c_id IN(SELECT c_id FROM Score WHERE s_id='01');

-- 12.查询和"01"号的同学学习的课程完全相同的其他同学的学号和姓名 


-- 13.把“SC”表中“张三”老师教的课的成绩都更改为此课程的平均成绩；

-- 14、查询没学过"张三"老师讲授的任一门课程的学生姓名
-- SELECT  s_name FROM Student AS stu, Score AS s WHERE c_id  IN (SELECT c_id FROM Course AS c, Teacher AS t WHERE t_name='张三' AND c.t_id=t.t_id) AND stu.s_id=s.s_id

-- 15、查询两门及其以上不及格课程的同学的学号，姓名及其平均成绩 
-- SELECT stu.s_id, s_name, avg_score FROM Student AS stu, 
--   (SELECT s_id, AVG(s_score) AS avg_score, COUNT(*) AS c_count FROM Score WHERE s_score < 60 GROUP BY s_id HAVING c_count>=2) AS new_s 	
-- WHERE stu.s_id=new_s.s_id;

-- 16、检索"01"课程分数小于60，按分数降序排列的学生信息
-- SELECT stu.*, new_s.s_score FROM Student AS stu, (SELECT s_id, s_score FROM Score WHERE c_id='01' AND s_score<60) AS new_s
-- WHERE stu.s_id=new_s.s_id ORDER BY new_s.s_score DESC;

-- 17、按平均成绩从高到低显示所有学生的所有课程的成绩以及平均成绩
-- SELECT  * FROM (SELECT s_id, AVG(s_score) FROM Score GROUP BY s_id) AS new_s, Score AS s WHERE new_s.s_id=s.s_id;	

-- 18.查询各科成绩最高分、最低分和平均分：以如下形式显示：课程ID，课程name，最高分，最低分，平均分，及格率，中等率，优良率，优秀率


-- 19.按各科平均成绩从低到高和及格率的百分数从高到低顺序
-- SELECT s.s_id, s.c_id, s_score,  FROM Score AS s
-- 
-- SELECT s_id, s_score, c_id FROM Score AS s WHERE s.s_score>=60; 

-- 20、查询学生的总成绩并进行排名
-- SELECT s_id, SUM(s_score) AS total_score FROM Score AS s GROUP BY s_id ORDER BY total_score DESC; 

-- 21、查询不同老师所教不同课程平均分从高到低显示 
-- SELECT s.s_id, s.c_id, c.t_id, AVG(s.s_score) AS avg_score FROM Score AS s, Course AS c WHERE c.c_id=s.c_id GROUP BY c.c_id ORDER BY avg_score DESC;

-- 22、查询所有课程的成绩第2名到第3名的学生信息及该课程成绩

-- 23、统计各科成绩各分数段人数：课程编号,课程名称,[100-85],[85-70],[70-60],[0-60]及所占百分比
-- 这道题 太复杂了懒的写


-- 24、查询学生平均成绩及其名次 # 如何添加序号
-- SELECT (@i:=@i+1) AS row_num, s.s_id, s.c_id, AVG(s.s_score) AS avg_score FROM  Score AS s, (Select @i:=0) AS b GROUP BY s.s_id ORDER BY avg_score DESC;


-- 25、查询各科成绩前三名的记录 ? 
-- 
-- SELECT s0.*, (SELECT COUNT(*) FROM Score AS s WHERE s.c_id=s0.c_id AND s.s_score>s0.s_score) + 1 rank
-- FROM Score AS s0
-- GROUP BY 1,2,3
-- HAVING rank<=3
-- ORDER BY s0.c_id, rank;

-- 26.查询每门课程被选修的学生数
-- SELECT s.c_id, COUNT(*) AS student_number FROM Score AS s GROUP BY s.c_id;

-- 27.查询出只选修了一门课程的全部学生的学号和姓名
-- SELECT stu.s_id, stu.s_name FROM Student AS stu,
--  (SELECT s_id, COUNT(*) FROM Score GROUP BY s_id HAVING COUNT(*)=1) AS new_s
-- WHERE stu.s_id=new_s.s_id;

-- 28、查询男生、女生人数 
-- SELECT COUNT(s_sex='男') AS boy_number, COUNT(s_sex='女') AS girl_number FROM Student;

-- 29、查询名字中含有"风"字的学生信息
-- SELECT * FROM Student WHERE s_name LIKE '%风%'

-- 30、查询同名同性学生名单，并统计同名人数 
-- SELECT s_name, COUNT(s_name) AS number FROM Student GROUP BY s_name HAVING COUNT(s_name)>=2;


-- 31、查询1990年出生的学生名单
-- SELECT * FROM Student WHERE s_brith LIKE '1990%';
-- 如果s_brith是DATE格式
-- SELECT * FROM Student WHERE `YEAR`(s_brith)='1990';
 
-- 32.查询每门课程的平均成绩，结果按平均成绩升序排列，平均成绩相同时，按课程号降序排列
-- SELECT c_id, AVG(s_score) AS avg_score FROM Score GROUP BY c_id ORDER BY avg_score ASC, c_id DESC;	

-- 37.查询不及格的课程，并按课程号从大到小排列 
-- SELECT c_id, s_score FROM Score WHERE s_score<60 ORDER BY c_id DESC;

-- 38.查询课程编号为"01"且课程成绩在60分以上的学生的学号和姓名；
-- SELECT stu.s_id, stu.s_name FROM Student AS stu, Score AS s WHERE s.c_id='01' AND s.s_score>60 AND stu.s_id=s.s_id;

-- 40.查询选修“张三”老师所授课程的学生中，成绩最高的学生姓名及其成绩
-- SELECT stu.s_name, new_s.max_score FROM Student AS stu, 
-- 	(SELECT s.s_id, MAX(s.s_score) AS max_score FROM Score AS s WHERE s.c_id IN (
-- 		SELECT c.c_id FROM Course AS c, Teacher AS t WHERE c.t_id=t.t_id AND t.t_name='张三'  
--  )) AS new_s
-- WHERE stu.s_id=new_s.s_id;

-- 42、查询每门功课成绩最好的前两名,存在并排分数的如何处理
-- SELECT s1.* FROM Score s1 WHERE
-- (
-- 	SELECT COUNT(1) FROM Score s2 WHERE
-- 	s1.c_id=s2.c_id AND s2.s_score>=s1.s_score
-- )<=2
-- ORDER BY s1.c_id,s1.s_score DESC

