---
layout: post
title: LeftJoinAndMapMany 성능개선 - Typeorm
subtitle: take, skip distinct alias 제거하기 
gh-repo: Jyoungho/Jyoungho.github.io
gh-badge: [star, fork, follow]
tags: [typeorm]
comments: true
---

## 😀 들어가며

배치 테스크에서 보내는 API 중 일부가 점차 지연이 발생되기 시작하였습니다. 해당 이슈는 처음에는 단순한 방편(timeout 시간 증가 등)으로 해결하려고 했으나, 문제가 지속적으로 발생하였고 이를 해결하고자 typeorm 쿼리를 분석해 보았습니다. 이를 통해서 leftJoinAndMapMany 를 사용할 때 주의해야될 점 성능을 향상시키는 방법에 대하여 공유하려고 합니다.

---

## 😧 문제사항

### 기존 Typeorm QueryBuilder

Typeorm 에서 조인하고 해당 데이터를 map 형식으로 반환받기 위하여 쿼리빌더의 leftJoinAndMapMany 을 사용하는 경우가 많습니다.

아래의 쿼리빌더가 실제로 문제가 발생했던 경우입니다.

``` javascript 
repository
.createQueryBuilder('group')
.innerJoinAndMapOne(
  'group.department',
  DepartMentEntity,
  'department',
  'group.id = departMent.groupId',
)
.leftJoinAndMapMany(
  'group.people',
  PersonDalEntity,
  'person',
  'group.id = person.groupId',
)
.where('group.status = :status', { status: ACTIVE })
.andWhere('group.createdAt BETWEEN :from AND :to', {
  from,
  to,
})
.orderBy('group.id', 'ASC')
.skip(startingAfter)
.take(limit)
.getMany();
```

위의 Typeorm 쿼리는 실제로 아래의 2개의 쿼리를 보낸다.

- 🎖️ Typorm 실제 쿼리

    ```sql
    SELECT DISTINCT `distinctAlias`.`group_id`
    FROM (
      SELECT 
          `group`.`id`                          AS `group_id`,
          `group`.`version`                     AS `group_versio`,
          `group`.`name`                        AS `group_name`,
          ...rows
      FROM `Group` `group`
            INNER JOIN `Department` `department` ON `group`.`id` = `department`.`groupId`
            LEFT JOIN `Person` `people` ON `group`.`id` = `deparment`.`groupId`
      WHERE `group`.`status` = 'ACTIVE'
        AND `group`.`createdAt` BETWEEN '2023-04-10T15:00:00.000' AND '2023-04-23T15:00:00.000'
    ) `distinctAlias` 
    ORDER BY `distinctAlias`.`group_id` ASC;
    ```

    ```sql
    SELECT * 
    FROM Group group 
        INNER JOIN Department department ON group . id = department . groupId
        LEFT JOIN Person person ON group . id = person . groupId 
    WHERE ( group . status = 'ACTIVE' AND group . nextResetDate BETWEEN '2023-04-10T15:00:00.000' AND '2023-04-23T15:00:00.000' ) 
      AND ( group . id IN ( ...ids ) )
    ORDER BY group . id ASC
    ```


typeorm 의 쿼리빌더로 작성된 실제쿼리는 하나지만 실제로 **2개의 쿼리**가 발생합니다.
첫 번째 쿼리가 약 **9초** 정도 소요되고, 2번째 쿼리는 금방 끝나는 쿼리라는 것을 알 수 있습니다.
하지만, **실제로 원하는 데이터는 2번째의 반환값입니다.**

### 초기에 예상한 문제점

- where condition 과 정확히 부합되는 **인덱스 부재**
    - where condition 에 포함된 인덱스를 생성해서 문제를 해결해보려고 했습니다.
    - 일부 속도의 개선은 있었지만 실제로 큰 변화는 없었습니다. (약 1s 정도 줄어듬)

- `select distinct` 로 인하여 발생하는 TempTable 로 발생하는 **Swap memory**
    - 컬럼 중에 json 데이터가 있어, TempTable 의 크기가 컸고 mysql 의 경우 기본적으로 임시테이블 설정이 16MB 으로 되어 있어 혹시 모를 swap memory 사용을 제한
      ( `tmp_table_size` , `max_heap_table_size` )

- `select distinct` 로 인하여 발생하는 TempTable 의 크기가 order by 절의 `sort_buffer_size` 크기보다 커서 **병합정렬**이 발생 (기본적으로 sort_buffer_size 는 256KB 입니다.)



### ❓ 원하지 않는 select distinct 는 왜 발생하는가?

typeorm 공식문서를 확인해보면 join 을 사용할 경우 take, skip 을 사용하지 않으면 의도치 않는 결과를 얻을 수 있다고 적혀있습니다.

> take and skip may look like we are using limit and offset, but they aren't. limit and offset may not work as you expect once you have more complicated queries with joins or subqueries. Using take and skip will prevent those issues.
>

leftJoinMapMany 와 offset, limit 을 사용할 경우 실제로 원하는 데이터를 얻지 못 할 수 있다.

---

## 🏆 해결방법

근본적으로 원하지 `DISTINCT` 을 typeorm 에서 보내기 때문에 해당부분을 제거하기 위하여 take, skip 대신에 offset, limit 을 활용하는 방식으로 쿼리빌더를 개선한다.

- take, skip을 사용할 경우 typeorm 의 쿼리
    - DISTINCT 를 통해 중복을 제거한 Id 들을 가져온다.
    - 그 후 WHERE IN condition 을 활용하여 해당 Id에 해당하는 값을 가져온다.
- 개선 후 `QueryBuilder`

    ``` sql
    getManager()
    .getRepository(GroupDalEntity)
    .createQueryBuilder('group')
    .select('group.id')
    .where('group.status = :status', { status: ACTIVE })
    .andWhere('group.createdAt BETWEEN :from AND :to', {
      from,
      to,
    })
    .orderBy('group.id', 'ASC')
    .offset(startingAfter)
    .limit(limit);
    ```

    ```sql
    getManager()
    .createQueryBuilder()
    .from((qb) => {
      return qb
        .select('g.*')
        .from(GroupDalEntity, 'g')
        .where(`g.id IN (:...groupIds)`, { groupIds });
    }, 'group')
    .select('group')
    .innerJoinAndMapOne(
      'group.department',
      DepartmentDalEntity,
      'department',
      'group.id = department.groupId',
    )
    .leftJoinAndMapMany(
      'group.people',
      PersonDalEntity,
      'person',
      'group.id = person.groupId',
    )
    .orderBy('group.id', 'ASC');
    ```


### 😎 결론적으로..

leftJoinAndMapMany 를 사용할 경우 아래와 같이 개선하면 성능을 향상시킬 수 있습니다.

- (filter) leftjoin 절과 group by 를 제거한 후 id 값들을 가져오는 쿼리
- (output) WHERE IN condition 을 활용한 쿼리

<br><br>
---
출처:
- "typeorm", https://orkhan.gitbook.io/typeorm/docs/select-query-builder