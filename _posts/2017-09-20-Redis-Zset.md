---
layout: post
title: "用Redis实现积分排行榜"
categories: Redis
tags: Zset
author: zch
---

* content
{:toc}
最近项目跟广州日报有个活动，需要在项目中增加一个根据学生的打卡获得的学分而生成的一个排行榜，考虑到访问量可能很大，且排行榜每天只更新一次，若每次都查看排行榜都查一次数据库，会导致服务器消耗很大，解决方案就是把每天的排行榜数据缓存到Redis的有序集合中，它会根据学分自动排序，满足了排行榜的需求。








## Redis Zset工具类

```java
public class ZsetRedisTemplate extends BaseRedisTemplate {

  public ZsetRedisTemplate(RedisTemplate<String, String> redisTemplate) {
    super(redisTemplate);
  }

  public void zadd(String key, String member, double score) {
    redisTemplate.opsForZSet().add(key, member, score);
  }

  public void zadd(String key, Set<ZSetOperations.TypedTuple<String>> set) {
    redisTemplate.opsForZSet().add(key, set);
  }

  public <T> List<T> zrevrangeByScore(String key, double min, double max, long offset, long count, Class<T> clazz) {
    List<T> results = new ArrayList<T>();
    Set<String> vals = redisTemplate.opsForZSet().reverseRangeByScore(key, min, max, offset, count);
    for(String val : vals) {
      T obj = JSON.parseObject(val, clazz);
      results.add(obj);
    }
    return results;
  }

  public Set<String> zrevrangeByScore(String key, double min, double max, long offset, long count) {
    return redisTemplate.opsForZSet().reverseRangeByScore(key, min, max, offset, count);
  }

  public List<String> zrevrangeByScoreForList(String key, double min, double max, long offset, long count) {

    Set<String> strs =  redisTemplate.opsForZSet().reverseRangeByScore(key, min, max, offset, count);
    if(strs.size() > 0) {
      List<String> result = new ArrayList<>();
      for(String str : strs) {
        result.add(str);
      }
      return result;
    } else {
      return null;
    }
  }

  public List<String> zrangeByScoreForList(String key, double min, double max) {
    Set<String> strs =  redisTemplate.opsForZSet().rangeByScore(key, min, max);
    if(strs.size() > 0) {
      List<String> result = new ArrayList<>();
      for(String str : strs) {
        result.add(str);
      }
      return result;
    } else {
      return null;
    }
  }

  public List<String> zrangeByScoreForList(String key, double min, double max, long offset, long count) {

    Set<String> strs =  redisTemplate.opsForZSet().rangeByScore(key, min, max, offset, count);
    if(strs.size() > 0) {
      List<String> result = new ArrayList<>();
      for(String str : strs) {
        result.add(str);
      }
      return result;
    } else {
      return null;
    }
  }

  public Set<String> zrevrangeByScore(String key, double min, double max) {
    return redisTemplate.opsForZSet().reverseRangeByScore(key, min, max);
  }

  public void zrem(String key, Object member) {
    redisTemplate.opsForZSet().remove(key, member);
  }

  public long zcard(String key, long min, long max) {
    return redisTemplate.opsForZSet().count(key, min, max);
  }

  public Double zscore(String key, String member) {
    return redisTemplate.opsForZSet().score(key, member);
  }

  public Long zrank(String key, String member) {
    return redisTemplate.opsForZSet().rank(key, member);
  }

  public void incrBy(String key, String member, Double num) {
    redisTemplate.opsForZSet().incrementScore(key, member, num);
  }

  public Long zrevrank(String key, String member) {
    return redisTemplate.opsForZSet().reverseRank(key, member);
  }
}
```



## 获取学生排行榜列表

```java
@Override
public Map<String, Object> scoreList(Long userId) {
  logger.info(">>>>>>>>>>>>>>>>>>>>>>>>> 获取学生学分榜 >>>>>>>>>>>>>>>>>>>>>>>>>>>");
  Map<String, Object> map = new MapBuilder().build();

  // 获取最近更新时间
  Long dayZeroTime;
  String timeStr = valueRedisTemplate.get(RedisConstants.RECENT_TIME);
  if (StringUtils.isNotBlank(timeStr)) {
    dayZeroTime = Long.valueOf(timeStr);
  } else {
    dayZeroTime = DateUtil.getDayZeroTime();
  }
  String date = DateUtil.format(new Date(dayZeroTime), DateUtil.NORMAL_DATETIME_FORMAT);

  List<UserScoreView> userScoreViewList = handleSort();
  UserScoreView userScore = null;
  if (userScoreViewList != null && userScoreViewList.size() > 0) {
    for (UserScoreView userScoreView : userScoreViewList) {
      if (userScoreView.getUserView().getUserId().equals(userId)) {
        userScore = userScoreView;
        break;
      }
    }
  }

  map.put("date", date);
  map.put("userScore", userScore);
  map.put("scoreList", userScoreViewList);
  return map;
}

private List<UserScoreView> handleSort() {
  String key = RedisConstants.CREDIT_RANKING;// 学生学分榜
  List<UserScoreView> userScoreViewList = new ArrayList<>();
  List<String> memberList = zsetRedisTemplate.zrevrangeByScoreForList(key, 0, Long.MAX_VALUE, 0, 200);
  if (memberList == null) {
    return null;
  }
  Long beforeSort = 0L;
  Long beforeScore = AttendanceScoreViewConverter.convert(Long.parseLong(memberList.get(0))).getScore();
  for (String id : memberList) {
    UserScoreView userScoreView = AttendanceScoreViewConverter.convert(Long.parseLong(id));
    Long sort = zsetRedisTemplate.zrevrank(key, id);
    Long score = zsetRedisTemplate.zscore(key, id).longValue();
    userScoreView.setScore(score);
    if (score.equals(beforeScore)) {
      userScoreView.setSort(beforeSort);
    } else {
      userScoreView.setSort(sort);
      beforeScore = score;
      beforeSort = sort;
    }
    userScoreViewList.add(userScoreView);
  }
  return userScoreViewList;
}
```



## 获取学校排行榜信息

```java
@Override
public Map<String, Object> schoolScoreList() {
  logger.info(">>>>>>>>>>>>>>>>>>>>>>>>> 获取学校学分榜 >>>>>>>>>>>>>>>>>>>>>>>>>>>");
  Map<String, Object> map = new MapBuilder().build();

  // 获取最近更新时间
  Long dayZeroTime;
  String timeStr = valueRedisTemplate.get(RedisConstants.RECENT_TIME);
  if (StringUtils.isNotBlank(timeStr)) {
    dayZeroTime = Long.valueOf(timeStr);
  } else {
    dayZeroTime = DateUtil.getDayZeroTime();
  }
  String date = DateUtil.format(new Date(dayZeroTime), DateUtil.NORMAL_DATETIME_FORMAT);
  List<SchoolScoreView> schoolScoreViewList = handleSchoolSort();
  map.put("date", date);
  map.put("scoreList", schoolScoreList);
  return map;
}

private List<SchoolScoreView> handleSchoolSort() {
  String key = RedisConstants.SCHOOL_CREDIT_RANKING;// 学校学分榜
  List<SchoolScoreView> schoolScoreViewList = new ArrayList<>();
  List<String> schoolNameList = zsetRedisTemplate.zrevrangeByScoreForList(key, 0, Long.MAX_VALUE, 0, 100);
  if (schoolNameList == null) {
    return null;
  }
  Long beforeSort = 0L;
  Long beforeScore = zsetRedisTemplate.zscore(key, schoolNameList.get(0)).longValue();
  for (String schoolName : schoolNameList) {
    SchoolScoreView schoolScoreView = new SchoolScoreView();
    Long sort = zsetRedisTemplate.zrevrank(key, schoolName);
    Long score = zsetRedisTemplate.zscore(key, schoolName).longValue();
    schoolScoreView.setSchoolName(schoolName);
    schoolScoreView.setScore(score);
    if (score.equals(beforeScore)) {
      schoolScoreView.setSort(beforeSort);
    } else {
      schoolScoreView.setSort(sort);
      beforeScore = score;
      beforeSort = sort;
    }
    schoolScoreViewList.add(schoolScoreView);
  }
  return schoolScoreViewList;
}
```



## 定时任务方法

```java
@Override
public void sqlLoadScore() {
  String userKey = RedisConstants.CREDIT_RANKING;// 学生学分榜
  String schoolKey = RedisConstants.SCHOOL_CREDIT_RANKING;// 学校学分榜

  Long dailyStartTime = Long.valueOf(valueRedisTemplate.get(RedisConstants.DAILY_START_TIME));// 10月9号
  Long dailyEndTime = Long.valueOf(valueRedisTemplate.get(RedisConstants.DAILY_END_TIME));// 10月18号
  if (dailyStartTime == null) {
    dailyStartTime = 1507478400000L;
  }
  if (dailyEndTime == null) {
    dailyEndTime = 1508256000000L;
  }
  Long now = System.currentTimeMillis();
  if (now > dailyStartTime && now < dailyEndTime) {
    try {
      // 广州日报用户ids
      if (!setRedisTemplate.exists(RedisConstants.DAILY_USER)) {
        List<MemberInfo> memberInfoList = memberInfoReposity.findList("SELECT * FROM member_info ", MemberInfo.class);
        for (MemberInfo memberInfo : memberInfoList) {
          setRedisTemplate.sadd(RedisConstants.DAILY_USER, String.valueOf(memberInfo.getCreateBy()), CommonConstants.Time.ONE_MONTH);
        }
      }
      Set<String> dailykUserIdSet = setRedisTemplate.member(RedisConstants.DAILY_USER);
      List<Long> dailyUserIds = new ArrayList<>();
      for (String id : dailykUserIdSet) {
        dailyUserIds.add(Long.parseLong(id));
      }

      // 删除旧的排行榜
      zsetRedisTemplate.del(userKey);
      zsetRedisTemplate.del(schoolKey);

      for (Long userId : dailyUserIds) {
        // 默认计算一个月， 从现在倒退算法计算
        for (int i = 0; i < 30; i++) {

          // 获取当天零点时间戳
          Long dayZeroTime = DateUtil.getDayZeroTime() - DateUtil.getTimeStampByHour(24 * i);
          logger.info("dayZeroTime -------- [" + dayZeroTime + "]");
          // 前一天零点时间戳
          Long beforeDayZeroTime = dayZeroTime - DateUtil.getTimeStampByHour(24 * (i + 1));
          logger.info("beforeDayZeroTime -------- [" + beforeDayZeroTime + "]");

          if (dayZeroTime > dailyStartTime && dayZeroTime <= dailyEndTime) {
            Integer maxScore = attendanceScoreRepository.findMaxScore(userId, beforeDayZeroTime, dayZeroTime);
            // 如果哪天没有打卡，以前的打卡记录就不计算
            if (null == maxScore) {
              break;
            }
            // 累加到缓存
            zsetRedisTemplate.incrBy(userKey, String.valueOf(userId), Double.parseDouble(String.valueOf(maxScore)));
            loadSchoolScore(userId, maxScore.longValue());
          }
        }
      }
      // 存储最近更新时间
      valueRedisTemplate.set(RedisConstants.RECENT_TIME, String.valueOf(System.currentTimeMillis()));
    } catch (Exception e) {
      logger.error(e.getMessage());
    }
  }
}
```

