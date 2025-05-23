---
title: 성적 관리 모델링 연습 (2)
description: 데이터베이스 중급(Modeling) - 성적 관리 모델링 연습 (2)
categories:
- Database
- 데이터베이스 중급(Modeling)

tags:
- Database

---

<!-- more -->

### 스토어드 프로시저 생성문 예시

```sql
CREATE PROCEDURE SP_Insert_SchoolClass
	-- ADD the parameters for the stored procedure here
	<@Param1, sysname, @p1> <Detatype_For_Param1, , int> = <Defualt_Value_For_Param1, , 0>,
	<@Param2, sysname, @p2> <Detatype_For_Param2, , int> = <Defualt_Value_For_Param2, , 0>
AS
BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets from
	-- interfering with SELECT statements
	SET NOCOUNT ON;
	
	-- INSERT statements for procedure here
	SELECT <@Param1, sysname, @p1>, <@Parama2, sysname, @p2>
END
```

### 반 테이블의 저장 프로시저 생성문
```sql
CREATE PROCEDURE SP_Insert_SchoolClass
	@SchoolGradeId tinyint,
	@SchoolClassId tinyint,
	@SchoolClassName varchar(50)
AS
BEGIN
	INSERT INTO TB_SchoolClass (SchoolGradeId, SchoolClassId, SchoolClassName)
	VALUES (@SchoolGradeId, @SchoolClassId, @SchoolClassName)
END
```

### 스토어드 프로시저 실행
```sql
exec SP_Insert_SchoolClass 1, 1, '1반'; -- ()괄호 필요없음
exec SP_Insert_SchoolCalss 1, 2, '2반';

SELECT * FROM TB_SchoolClass;
```

|  | SchoolGradeId | SchoolClassId | SchoolClassName |
| --- | --- | --- | --- |
| 1 | 1 | 1 | 1반 |
| 2 | 1 | 2 | 2반 |

### 반 테이블의 업데이트 프로시저

- SP_Update_SchoolClassName
- SP_Update_UseFlag

### 반 테이블의 조회 프로시저

- SP_SchoolClass_GetById(gradeId, classId);
- SP_SchoolClass_GetAll();
- SP_SchoolClass_GetByGrade(gradeId);

<figure align="center">
<img src="/post_images/Database/grade-level-student-rel.png">
<figcaption>학년-반-학생 관계도</figcaption>
</figure>

- 신입생의 경우 아직 반 배정을 받지 못했지만 학생 테이블에 등록해야 되는 경우를 위해서 `SchoolClassId` 와 `StudentSeqNo`는 NULL 허용을 해준다.
- `SchoolGradeId` , `SchoolClassId` , `SchoolSeqNo` 를 묶어서 후보키로 설정하여 NULL은 허용하지만 UNIQUE 를 만족시키기 때문에 반과 반별 학생 번호의 중복 생성 체계가 가능해진다.
- 학생 테이블에 `entrance_date` 입학날짜와 `graduate_date` 졸업날짜를 속성으로 설정 가능하다. 입학날짜가 없는 학생은 없으므로 NOT NULL 이지만, 아직 졸업하지 못한 학생들은 졸업날짜가 NULL이다. 만약 졸업날짜에 날짜 데이터가 들어가 있으면 졸업생인 것을 인지할 수 있다.
- 반 테이블의 `UseFlag` 는 현재 년도에 해당 반을 사용하지 않는다는 정보를 파악하기 위한 속성이다. 사용하지 않는다고 해당 반 테이블을 삭제할 수는 없다. 해당 반에 배정되어 있던 학생들이 부모가 없는 자식이 되기 때문에, 삭제 되신 UseFlag 의 업데이트문을 사용하여 반의 사용여부를 결정하는 것이 좋다.

---
