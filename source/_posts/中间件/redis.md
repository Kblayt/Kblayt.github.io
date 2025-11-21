---
title: redis
date: 2022-10-29 13:05:13
categories:
    - 中间件
tags:
    - redis
---
# redis安装部署与启动

https://www.runoob.com/redis/redis-install.html

# redis的简单调用



```
package com.example.springboot_redis;

import com.alibaba.fastjson.JSON;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.redis.core.HashOperations;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;

import javax.annotation.Resource;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@SpringBootTest
class SpringbootRedisApplicationTests {


    @Resource
    private RedisTemplate redisTemplate;

    @Test
    public void setKV() {
        ValueOperations valueOperations = redisTemplate.opsForValue();
        valueOperations.set("姓名","Kblayt",1, TimeUnit.SECONDS);
    }

    @Test
    public void getKV() {
        ValueOperations valueOperations = redisTemplate.opsForValue();
        String name = (String) valueOperations.get("姓名");
        System.out.println(name);
    }

    @Test
    public void setKMap() {
        ValueOperations valueOperations = redisTemplate.opsForValue();
        HashOperations hashOperations = redisTemplate.opsForHash();
        Map map = new HashMap();
        map.put("姓名","Kblayt");
        map.put("性别","男");
        map.put("年龄","19");
        String JsonMap = JSON.toJSONString(map);
        hashOperations.putAll("参与人员1",map);
        valueOperations.set("参与人员2",JsonMap);
    }

    @Test
    public void getKMap1() {
        HashOperations hashOperations = redisTemplate.opsForHash();
        Map msg = hashOperations.entries("参与人员1");
        System.out.println(msg);
    }

    @Test
    public void getKMap2() {
        ValueOperations valueOperations = redisTemplate.opsForValue();
        Object msg = valueOperations.get("参与人员2");
        System.out.println(msg.toString());
    }

}

```
redis传输乱码配置
```
package com.takeaway.merchantsprovider.config;


import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.net.UnknownHostException;

@Configuration
public class RedisConfig {
    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
            throws UnknownHostException {

        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        template.setKeySerializer(stringRedisSerializer);
        template.setValueSerializer(stringRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }
}

```

