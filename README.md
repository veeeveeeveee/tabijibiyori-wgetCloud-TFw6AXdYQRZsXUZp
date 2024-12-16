
**大纲**


**1\.社区电商购物车的读多写多场景分析**


**2\.购物车的复杂缓存与异步落库(Sorted Set \+ Hash \-\> hPut \+ zadd)**


**3\.购物车异步落库与完整加入流程(缓存雪崩 \+ MQ异步出现问题)**


**4\.购物车的阈值检查与重复加入逻辑(hGet \+ hLen \+ hFieldExists)**


**5\.购物车加入商品多线程并发问题解决(分布式锁保证请求幂等)**


**6\.购物车的查询、更新功能(zrevrange \+hGetAll \+ zremove\+ hDel)**


**7\.购物车的选中提交功能**


**8\.简单总结**


 


**1\.社区电商购物车的读多写多场景分析**


**(1\)对用户数据和分享贴列表数据的处理**


**(2\)对购物车数据的处理**


 


**(1\)对用户数据和分享贴列表数据的处理**


在新增或修改时，采用的是同步写库\+写缓存或异步写库\+写缓存的方案。其中用户数据缓存会进行同步更新，而分享贴列表数据的分页缓存则由于可能需要重建的分页缓存比较多，则会通过异步更新。并且在读取用户数据、分享贴列表数据的时候，是直接读缓存的。除非是读到了被淘汰掉的冷数据，才会重新读数据库 \+ 写缓存。


 


之所以这样设计，是因为用户数据和分享贴数据都是典型读多写少的数据，可能用户数据是0\.01%写 \+ 99\.99%读，而分享贴数据是1%写 \+ 99%读。由于写的情况很少，所以对应数据库的写压力也就很小。因此在用户数据写库时直接采用同步写库和写缓存，是没问题的。由于读的情况很多，所以通过先读缓存就可以用缓存抗下大量高并发的读。


 


**(2\)对购物车数据的处理**


首先购物车功能包括：加入购物车、查看购物车、编辑购物车、发起结算等。然后当平台进行促销活动时，比如发起一些种草商品的团购活动。这时购物车的数据就会变成读多写多的数据，此时的购物车的数据可能会出现高并发的写。如果购物车数据的写也同步落库，那么可能就会导致数据库的压力很大。


 


因此对于购物车或者库存这种读多写多的数据，由于存在大量高并发的写、大量高并发的读，那么我们会把主要数据基于Redis来进行主存储，来实现高性能读写，同时通过异步把数据同步到MySQL进行持久化落库。


 


**2\.购物车的复杂缓存与异步落库(Sorted Set \+ Hash \-\> hPut \+ zadd)**


由于购物车的数据是读多写多的数据，所以会使用缓存来存储主数据，以便能够抗住高并发的写和读，然后进行落库的时候再通过异步来落库。此外，商品系统一般也会使用缓存架构来提供商品数据的读接口。


 


更新购物车时，需要涉及如下操作：


一.更新用户购物车的SKU数量缓存


二.更新加入到购物车的SKU扩展信息缓存


三.更新用户购物车的SKU加购时间排序缓存


 


由于购物车包含很多种数据，所以会分成多个缓存key，这些key对应的缓存数据类型有：


Hash数据类型：购物车商品数量和信息的哈希{skuId \-\> count}和{skuId \-\> skuInfo}


ZSet数据类型：购物车加购排序的有序集合\[skuId \-\> timestamp]



```
@Service
public class CookBookCartServiceImpl implements CookBookCartService {
    ...
    @Transactional(rollbackFor = Exception.class)
    @Override
    public void addCartGoods(AddCookBookCartRequest request) {
        //构造购物车商品数据DTO
        CartSkuInfoDTO cartSkuInfoDTO = buildCartSkuInfoDTO(request);
        //校验商品是否可售: 库存、上下架状态、购物车是否达到最大限制(购物车sku数量最多不能超过100)
        checkSellableProduct(cartSkuInfoDTO);
        //检查加购SKU是否已经存在购物车中
        if (checkCartSkuExist(cartSkuInfoDTO)) {
            //重新计算数量
            cartSkuInfoDTO = recalculateQuantity(cartSkuInfoDTO);
            //更新购物车Redis缓存
            updateCartCache(cartSkuInfoDTO);
            //发送更新购物车SKU的消息到MQ
            sendAsyncUpdateMessage(cartSkuInfoDTO);
            return;
        }
        //更新缓存
        //购物车的缓存数据类型有：hash:{skuId->count}, hash:{skuId->skuInfo}, zset:[skuId->timestamp]
        updateCartCache(cartSkuInfoDTO);
        //发送新增购物车SKU的消息到MQ
        sendAsyncPersistenceMessage(cartSkuInfoDTO);
    }

    //更新购物车缓存
    private void updateCartCache(CartSkuInfoDTO cartSkuInfoDTO) {
        //更新用户购物车的SKU数量缓存
        updateCartNumCache(cartSkuInfoDTO);
        //更新加入到购物车的SKU扩展信息缓存
        updateCartExtraCache(cartSkuInfoDTO);
        //更新用户购物车的SKU加购时间排序缓存
        updateCartSortCache(cartSkuInfoDTO);
    }

    //更新用户购物车的SKU数量缓存
    private void updateCartNumCache(CartSkuInfoDTO cartSkuInfoDTO) {
        //用户购物车SKU数量缓存的key
        String numKey = RedisKeyConstant.SHOPPING_CART_HASH + cartSkuInfoDTO.getUserId();
        Integer count = cartSkuInfoDTO.getCount();
        String field = cartSkuInfoDTO.getSkuId();
        //更新用户购物车的SKU数量缓存
        //选用Redis里的Hash数据结构
        //shopping_cart_hash_userId : {
        //   skuId1: 2,
        //   skuId2: 3
        //}}
        redisCache.hPut(numKey, field, String.valueOf(count));
        log.info("更新用户购物车的SKU数量缓存, key: {}, field: {}, value: {}", numKey, field, count);
    }

    //更新加入到购物车的SKU扩展信息缓存
    private void updateCartExtraCache(CartSkuInfoDTO cartSkuInfoDTO) {
        //缓存加入到购物车的SKU扩展信息的key
        String extraKey = RedisKeyConstant.SHOPPING_CART_EXTRA_HASH + cartSkuInfoDTO.getUserId();
        String field = cartSkuInfoDTO.getSkuId();
        //更新加入到购物车的SKU扩展信息缓存
        //选用Redis里的Hash数据结构
        //shopping_cart_extra_hash_userId : {
        //  skuId1: skuInfo1,
        //  skuId2: skuInfo2
        //}
        redisCache.hPut(extraKey, field, JsonUtil.object2Json(cartSkuInfoDTO));
        log.info("更新加入到购物车的SKU扩展信息缓存, key: {}, field: {}, value: {}", extraKey, field, JsonUtil.object2Json(cartSkuInfoDTO));
    }

    //更新用户购物车的SKU加购时间排序缓存
    private void updateCartSortCache(CartSkuInfoDTO cartSkuInfoDTO) {
        //缓存用户购物车里的SKU加购时间排序的key
        //Redis的五大数据结构：String、List、Set、Sort Set、Hash
        //其中Sorted Set可以对写入的数据给一个score分数，Sorted Set会默认按照score来进行排序
        String sortKey = RedisKeyConstant.SHOPPING_CART_ZSET + cartSkuInfoDTO.getUserId();
        String field = cartSkuInfoDTO.getSkuId();
        //下面把每个skuId和它加入购物车的时间，写入到Sorted Set里去
        //Sorted Set: [{skuId1 -> score1(当前时间)}, {skuId2 -> score2(当前时间)}]
        redisCache.zadd(sortKey, field, System.currentTimeMillis());
        log.info("更新用户购物车的SKU加购时间排序缓存, key: {}", sortKey);
    }
}
```

 


**3\.购物车异步落库的消息丢失与不一致分析(缓存雪崩 \+ MQ异步出现问题)**


**(1\)异步落库时缓存崩了没有出现数据不一致(可以通过降级 \+ 缓存预热加载恢复)**


**(2\)异步落库时缓存写了但是MQ没有写成功从而出现数据不一致**


**(3\)先出现不一致然后缓存崩了从而造成数据丢失**


**(4\)为什么购物车的主数据存储要选用Redis**


 


**(1\)异步落库时缓存崩了没有出现数据不一致(可以通过降级 \+ 缓存预热加载恢复)**


此时其实就是缓存雪崩的情况。


 


现已知用户在加购一个商品SKU到购物车时，会进行异步化落库磁盘。这时MySQL数据库有点像是备用存储，主要用在异步同步数据时备份数据。


 


一般来说购物车的主数据存储，是由Redis来实现的，并都优先从Redis中进行购物车的写和读，这时是不会有不一致的问题的。


 


由于落库时通过异步化使用MySQL备用存储，那么万一Redis集群全都崩溃了，这时可能就会导致购物车的主数据都没了。


 


但即便Redis主数据全都没了，我们还是可以基于MySQL来进行降级，通过降级继续提供购物车的写和读。然后等缓存恢复后，再进行缓存预热加载，把数据库里的数据加载到缓存里。当然缓存集群崩了，其实就是缓存雪崩的问题了。


 


**(2\)异步落库时缓存写了但是MQ没有写成功从而出现数据不一致**


此时其实就是MQ异步出现问题的情况。


 


此时可能会对应下面两种异常情况：


情况一：刚刚写完缓存，还没来得及发送消息到MQ里，突然系统崩了。导致缓存写成功，但是异步消息没有发送出去


情况二：刚刚写完缓存，系统正常运行，已经向MQ发出消息，但RocketMQ崩了。导致缓存写成功，但是异步消息也没发送成功


 


这两种异常情况都会导致Redis里有数据，但RocketMQ里没消息。这时Redis中的数据自然也就没有异步落库到MySQL，从而造成Redis缓存和MySQL数据库之间的数据是不一致的。


 


这时候其实问题也不太大，即使出现了这样的情况，只要Redis里有数据就即可。因为用户后续一旦提交购物车生成订单，那么其数据就会从Redis里删除，这时MySQL就会跟Redis同步了。


 


**(3\)先出现不一致然后缓存崩了从而造成数据丢失**


如果是Redis突然崩溃了，导致只有MySQL里有数据了。但是MySQL之前又因MQ崩了丢了一条数据，那么此时因为Redis崩了所以那条数据就丢失了。


 


这时其实问题也不大，因为这最多导致用户在购物车里找不到自己刚加入的商品。而且购物车只要没发起提交，Redis的本质还是临时性的数据存储空间。在购物车中找不到商品，那么重新加入购物车即可。


 


**(4\)为什么购物车的主数据存储要选用Redis**


因为从业务上来说，购物车的数据其实是属于临时性的数据，用户仅仅是把一些商品在购物车里进行暂存。对用户来说，购物车里的商品会有三种情况：


一.不发起购买，从购物车里直接删除这些商品


二.过了很长时间都没买，用户都已经把它给忘了


三.选择购物车里的商品发起购买


 


所以对于这种比较偏临时的数据，使用Redis来当主数据的存储是没问题的。即便出现缓存雪崩或MQ异步出现问题，导致Redis和MySQL数据不一致，甚至购物车数据丢失，那么问题也不大。因为极端情况下，购物车少了一些商品，大不了让用户重新加购。


 


下面是新增用户购物车和更新用户购物车时的异步落库相关代码：



```
@Service
public class CookBookCartServiceImpl implements CookBookCartService {
    ...
    //发送新增购物车SKU的消息到MQ
    private void sendAsyncPersistenceMessage(CartSkuInfoDTO cartSkuInfoDTO) {
        //需要落库的购物车实体对象
        CookBookCartDO cartDO = cookbookCartConverter.dtoToDO(cartSkuInfoDTO);

        //发送消息到MQ
        log.info("发送新增购物车SKU的消息到MQ, topic: {}, skuInfo: {}", RocketMqConstant.COOKBOOK_ASYNC_PERSISTENCE_MESSAGE_SEND_TOPIC, JsonUtil.object2Json(cartDO));
        defaultProducer.sendMessage(RocketMqConstant.COOKBOOK_ASYNC_PERSISTENCE_MESSAGE_SEND_TOPIC, JsonUtil.object2Json(cartDO), "COOKBOOK购物车异步落库消息");
    }
    
    //发送更新购物车SKU的消息到MQ
    private void sendAsyncUpdateMessage(CartSkuInfoDTO cartSkuInfoDTO) {
        //需要落库的购物车实体对象
        CookBookCartDO cartDO = cookbookCartConverter.dtoToDO(cartSkuInfoDTO);

        //发送消息到MQ
        log.info("发送更新购物车SKU的消息到MQ, topic: {}, cartSkuInfo: {}", RocketMqConstant.COOKBOOK_ASYNC_UPDATE_MESSAGE_SEND_TOPIC, JsonUtil.object2Json(cartDO));
        defaultProducer.sendMessage(RocketMqConstant.COOKBOOK_ASYNC_UPDATE_MESSAGE_SEND_TOPIC, JsonUtil.object2Json(cartDO), "COOKBOOK购物车异步更新消息");
    }
}

@Component
public class CookbookCartPersistenceListener implements MessageListenerConcurrently {
    @Autowired
    private CookBookCartDAO cookBookCartDAO;

    //消费新增购物车SKU的消息
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List msgList, ConsumeConcurrentlyContext context) {
        try {
            for (MessageExt messageExt : msgList) {
                log.info("执行购物车加购落库消息逻辑，消息内容：{}", messageExt.getBody());
                String msg = new String(messageExt.getBody());
                CookBookCartDO cartDO = JSON.parseObject(msg, CookBookCartDO.class);

                //用户在加购一个商品SKU到购物车时，实行的是异步化落库
                //所以MySQL数据库的作用就有点类似于备用存储了
                //一般来说购物车的主数据存储，会通过Redis来实现，并都优先对Redis进行写和读，这时是不会有不一致的问题的
                //由于落库时通过异步化使用MySQL备用存储，那么万一Redis集群全都崩溃了，这时可能就会导致购物车的主数据都没了
                //此时可以基于MySQL数据库来进行降级，降级提供购物车的写和读
                //等缓存恢复了以后，再进行缓存预热加载，这时数据库里的数据再加载到缓存里去
                log.info("购物车数据开始保存到MySQL，userId: {}, cartDO: {}", cartDO.getUserId(), msg);
                cookBookCartDAO.save(cartDO);
            }
        } catch (Exception e) {
            log.error("consume error, 购物车落库消息消费失败", e);
            //本次消费失败，下次重新消费
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
        log.info("购物车加购持久化消息消费成功, result: {}", ConsumeConcurrentlyStatus.CONSUME_SUCCESS);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}

@Component
public class CookbookCartUpdateListener implements MessageListenerConcurrently {
    @Autowired
    private CookBookCartDAO cookBookCartDAO;

    //消费更新购物车SKU的消息
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List msgList, ConsumeConcurrentlyContext context) {
        try {
            for (MessageExt messageExt : msgList) {
                log.info("执行购物车持久化更新消息逻辑，消息内容：{}", messageExt.getBody());
                String msg = new String(messageExt.getBody());
                CookBookCartDO cartDO = JSON.parseObject(msg, CookBookCartDO.class);

                log.info("购物车数据开始更新到MySQL，userId: {}, cartDO: {}", cartDO.getUserId(), msg);
                UpdateWrapper<CookBookCartDO> updateWrapper = new UpdateWrapper<>();
                updateWrapper.set("count", cartDO.getCount());
                updateWrapper.eq("user_id", cartDO.getUserId());
                updateWrapper.eq("sku_id", cartDO.getSkuId());
                if (cartDO.getCount() == 0) {
                    cookBookCartDAO.remove(updateWrapper);
                    continue;
                }
                cookBookCartDAO.update(updateWrapper);
            }
        } catch (Exception e) {
            // 本次消费失败，下次重新消费
            log.error("consume error, 购物车持久化更新消息消费失败", e);
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
        log.info("购物车更新持久化消息消费成功, result: {}", ConsumeConcurrentlyStatus.CONSUME_SUCCESS);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}
```

 


**4\.购物车的阈值检查与重复加入逻辑(hGet \+ hLen \+ hFieldExists)**


**(1\)hLen命令获取当前购物车sku数量**


**(2\)hFieldExists检查商品是否存在**


**(3\)加购时候判断是否重复加入**


 


主要是使用Redis的Hash数据类型的几个命令进行检查，比如hLen获取Hash数据类型某个key下的元素个数，比如hFieldExists判断在Hash数据类型中是否存在某个key。


 


**(1\)hlen命令获取当前购物车sku数量**



```
//校验购物车商品数量是否超过阈值
private void checkCartProductThreshold(Long userId) {
    //从缓存中(Hash数据类型)获取当前购物车sku数量
    //hash: { skuId1 -> count1, skuId2 -> count2 }，hLen命令可以拿到某个key对应的Hash结构里有多少个数据条目
    //购物车里能加入多少商品是有限制的
    Long len = redisCache.hLen(RedisKeyConstant.SHOPPING_CART_HASH + userId);
    if (len >= CookbookCartConstants.CART_DEFAULT_MAX_SKU_COUNT) {
        throw new CookbookCartBizException(CookbookCartErrorCodeEnum.CART_SKU_COUNT_THRESHOLD_ERROR, CookbookCartErrorCodeEnum.CART_SKU_COUNT_THRESHOLD_ERROR.getErrorCode());
    }
}
```

**(2\)hFieldExists检查商品是否存在**



```
//检查加购SKU是否已经存在购物车中
private boolean checkCartSkuExist(CartSkuInfoDTO cartSkuInfoDTO) {
    //hash: { skuId1 -> skuInfo1, skuId2 -> skuInfo2 }
    return redisCache.hFieldExists(RedisKeyConstant.SHOPPING_CART_EXTRA_HASH + cartSkuInfoDTO.getUserId(), cartSkuInfoDTO.getSkuId());
}
```

**(3\)加购时候判断是否重复加入**


如果某个商品SKU是重复加入的，那么就要：重新计算数量、更新购物车的Redis缓存、发送更新消息到MQ。



```
//检查加购SKU是否已经存在购物车中
if (checkCartSkuExist(cartSkuInfoDTO)) {
    //重新计算数量
    cartSkuInfoDTO = recalculateQuantity(cartSkuInfoDTO);
    //更新购物车Redis缓存
    updateCartCache(cartSkuInfoDTO);
    //发送更新消息到MQ
    sendAsyncUpdateMessage(cartSkuInfoDTO);
    return;
}

//检查加购的商品SKU是否在购物车中存在，如果存在，那么商品数量+1
private CartSkuInfoDTO recalculateQuantity(CartSkuInfoDTO cartSkuInfoDTO) {
    //获取缓存中存在的商品数据
    String cartSkuInfoString = redisCache.hGet(RedisKeyConstant.SHOPPING_CART_EXTRA_HASH + cartSkuInfoDTO.getUserId(), cartSkuInfoDTO.getSkuId()).toString();
    CartSkuInfoDTO oldCartSkuInfoDTO = JsonUtil.json2Object(cartSkuInfoString, CartSkuInfoDTO.class);
    if (Objects.isNull(oldCartSkuInfoDTO)) {
        return cartSkuInfoDTO;
    }

    //商品数量加1
    oldCartSkuInfoDTO.setCount(oldCartSkuInfoDTO.getCount() + 1);
    return oldCartSkuInfoDTO;
}
```

 


**5\.购物车加入商品多线程并发问题解决(分布式锁保证请求幂等)**


如果某个用户在加购时连续点了几次，那么由于网络等原因，就可能出现这几次请求并发到达服务器端来进行处理，导致重复加购问题。



```
//检查加购的商品是否在购物车中存在，如果存在，那么商品数量 + 1
private CartSkuInfoDTO recalculateQuantity(CartSkuInfoDTO cartSkuInfoDTO) {
    //获取缓存中存在的商品数据
    String cartSkuInfoString = redisCache.hGet(RedisKeyConstant.SHOPPING_CART_EXTRA_HASH + cartSkuInfoDTO.getUserId(), cartSkuInfoDTO.getSkuId()).toString();
    CartSkuInfoDTO oldCartSkuInfoDTO = JsonUtil.json2Object(cartSkuInfoString, CartSkuInfoDTO.class);
    if (Objects.isNull(oldCartSkuInfoDTO)) {
        return cartSkuInfoDTO;
    }

    //商品数量加1
    //如果一个用户连续点击了3次要加入同一个商品，这时可能由于网络原因会出现3个线程一起来进行加购操作
    //这3个线程会一起把一条商品数据读出来，此时每个线程读到的购买数量都是1
    //然后3个线程都会把这个购买数量+1，由于Redis操作单线程，最后购买数量变成：1+1+1+1=4
    oldCartSkuInfoDTO.setCount(oldCartSkuInfoDTO.getCount() + 1);
    return oldCartSkuInfoDTO;
}
```

可以通过在加入购物车的方法入口添加分布式锁来解决这个问题，也就是加分布式锁保证请求幂等性。



```
@Transactional(rollbackFor = Exception.class)
@Override
public void addCartGoods(AddCookBookCartRequest request) {
    String updateCartLockKey = RedisLockKeyConstants.UPDATE_CART_LOCK_KEY + request.getUserId() + ":" + request.getSkuId();
    boolean locked = redisLock.blockedLock(updateCartLockKey);

    if (!locked) {
        throw new BaseBizException("商品加入购物车失败");
    }
    
    try {
        ...
    } finally {
        redisLock.unlock(RedisLockKeyConstants.UPDATE_CART_LOCK_KEY);
    }
}
```

 


**6\.购物车的查询、更新与选中功能(zrevrange \+hGetAll \+ zremove\+ hDel)**


**(1\)购物车的查询流程(基于有序集合 \+ 哈希来查)**


**(2\)从缓存中获取购物车数据(基于zrevrange倒序 \+ hGetAll来查)**


**(3\)通过分布式锁从数据库中获取购物车数据**


**(4\)购物车的更新(基于zremove \+ hDel来删除)**


 


**(1\)购物车的查询流程(基于有序集合 \+ 哈希来查)**


一.如果缓存中存在，那么就从缓存中查询到后返回


二.如果缓存中不存在，那么就首先添加分布式锁，然后再查询MySQL，查询到数据后便将数据更新到缓存，最后返回


三.如果缓存和MySQL都不存在，那么就在查询MySQL后，缓存一个空值并设置随机过期时间。当下次再来查询购物车时，会先判断缓存中的空值是否存在，如果存在就不查数据库了



```
@RestController
@RequestMapping("/api/goodscart")
public class CookBookCartController {
    ...
    //查询购物车入口
    @RequestMapping("/queryCart")
    public JsonResult queryCart(Long userId) {
        //购物车缓存是使用Redis的Sorted Set + Hash来实现的
        //因此会先查询按时间排序的商品skuId集合，然后再查询每个skuId对应商品信息
        CookBookCartInfoDTO dto = cookBookCartService.queryCart(userId);
        return JsonResult.buildSuccess(dto);
    }
}

@Service
public class CookBookCartServiceImpl implements CookBookCartService {
    ...
    @Override
    public CookBookCartInfoDTO queryCart(Long userId) {
        //1.检查入参
        checkParams(userId);
        //2.从缓存中获取购物车数据
        CookBookCartInfoDTO cartDTO = queryCartByCache(userId);
        //3.如果缓存中没有就从数据库中获取购物车数据
        return Objects.nonNull(cartDTO) ? cartDTO : queryCartNoCache(userId);
    }
}
```

**(2\)从缓存中获取购物车数据(基于zrevrange倒序 \+ hGetAll来查)**


由于购物车缓存是使用Redis的Sorted Set \+ Hash来实现的，因此会先查询按时间排序的商品skuId集合，再查询每个skuId对应信息，通过zrevrange倒序和hGetAll查出购物车商品集合：



```
//从缓存中获取购物车数据
private CookBookCartInfoDTO queryCartByCache(Long userId) {
    //从缓存中查询出有序的购物车商品集合
    List<CartSkuInfoDTO> totalSkuList = getCartInfoDTOFromCache(userId);
    //如果缓存中没有就返回null
    if (totalSkuList.size() == 0) {
        return null;
    }
    //未失效的商品列表
    List<CartSkuInfoDTO> skuList = new ArrayList<>();
    //失效的商品列表
    List<CartSkuInfoDTO> disabledSkuList = new ArrayList<>();
    //拆分购物车商品列表为：失效的商品列表、未失效的商品列表
    splitCartSkuList(totalSkuList, skuList, disabledSkuList);
    //根据未失效的商品列表计算结算价格
    BillingDTO billingDTO = calculateCartPriceByCoupon(userId, skuList);
    //返回购物车数据结构
    return CookBookCartInfoDTO.builder().skuList(skuList).disabledSkuList(disabledSkuList).billing(billingDTO).build();
}

//从缓存中查询出有序购物车商品集合
@SuppressWarnings("unchecked")
private List<CartSkuInfoDTO> getCartInfoDTOFromCache(Long userId) {
    //从缓存中获取有序的商品ID列表
    
    //把Sorted Set里所有的数据都查出来，写入数据的时候默认就已经排序过了
    Set<String> orderSkuIds = redisCache.zrevrange(RedisKeyConstant.SHOPPING_CART_ZSET + userId, CookbookCartConstants.ZSET_ALL_RANGE_START_INDEX, CookbookCartConstants.ZSET_ALL_RANGE_END_INDEX);

    //从缓存中获取商品扩展信息
    Map<String, String> cartInfo = redisCache.hGetAll(RedisKeyConstant.SHOPPING_CART_EXTRA_HASH + userId);

    //遍历有序的商品ID列表，获取商品信息集合
    return orderSkuIds.stream().filter(StringUtils::isNotEmpty).filter(skuId -> !skuId.equals(CookbookCartConstants.EMPTY_CACHE_IDENTIFY)).map(skuId -> JsonUtil.json2Object(cartInfo.get(skuId), CartSkuInfoDTO.class)).collect(Collectors.toList());
}
```

**(3\)通过分布式锁从数据库中获取购物车数据**


这里加分布式锁，主要是为了保护数据库。



```
//从数据库中获取购物车数据
private CookBookCartInfoDTO queryCartNoCache(Long userId) {
    //判断是否存在空缓存
    String emptyKey = RedisKeyConstant.SHOPPING_CART_EMPTY + userId;
    if (redisCache.hasKey(emptyKey)) {
        log.warn("购物车查询到空缓存，禁止查询MySQL, key: {}", emptyKey);
        return CookBookCartInfoDTO.builder().build();
    }

    List<CartSkuInfoDTO> cartInfoDTOs;
    String key = RedisLockKeyConstants.SHOPPING_CART_PERSISTENCE_KEY + userId;
    try {
        boolean lock = redisLock.lock(key);
        if (!lock) {
            throw new CookbookCartBizException(CookbookCartErrorCodeEnum.CART_PERSISTENCE_ERROR, CookbookCartErrorCodeEnum.CART_PERSISTENCE_ERROR.getErrorCode());
        }
        log.warn("购物车缓存数据查询为空, 添加分布式锁查询MySQL, userId: {}", userId);
        //从数据库中查询到购物车的商品集合
        cartInfoDTOs = getCartDTOFromPersistence(userId);
        //更新Redis缓存中的购物车商品
        syncCacheFromPersistence(userId, cartInfoDTOs);
    } finally {
        redisLock.unlock(key);
    }
    //构造购物车返回值
    return buildCookbookCartInfoDTO(userId, cartInfoDTOs);
}
```

**(4\)购物车的更新(基于zremove \+ hDel来删除)**


更新购物车其实主要是更新SKU数量，也要加分布式锁保证请求幂等性。更新购物车数量为0，即删除缓存时，就使用zremove和hDel。



```
@RestController
@RequestMapping("/api/goodscart")
public class CookBookCartController {
    ...
    //更新购物车，主要是更新购物车的SKU数量
    @RequestMapping("/updateCartGoods")
    public JsonResult updateCartGoods(@RequestBody @Valid UpdateCookBookCartRequest request){
        CookBookCartInfoDTO dto = cookBookCartService.updateCartGoods(request);
        return JsonResult.buildSuccess(dto);
    }
}

@Service
public class CookBookCartServiceImpl implements CookBookCartService {
    ...
    @Transactional(rollbackFor = Exception.class)
    @Override
    public CookBookCartInfoDTO updateCartGoods(UpdateCookBookCartRequest request) {
        String updateCartLockKey = RedisLockKeyConstants.UPDATE_CART_LOCK_KEY + request.getUserId() + ":" + request.getSkuId();
        boolean locked = redisLock.blockedLock(updateCartLockKey);
        if (!locked) {
            throw new BaseBizException("商品加入购物车失败");
        }

        try {
            //获取购物车中的商品
            CartSkuInfoDTO cartSkuInfoDTO = getCartSkuInfoDTO(request);
            //校验商品可售状态：库存
            checkSellableProduct(cartSkuInfoDTO);
            if (request.getCount() == 0) {
                //删除商品缓存
                clearCartCache(cartSkuInfoDTO);
                //发MQ持久化到MySQL
                sendAsyncUpdateMessage(cartSkuInfoDTO);
                //返回空数据
                return CookBookCartInfoDTO.builder().build();
            }
            //更新缓存
            updateCartCache(cartSkuInfoDTO);
            //发MQ持久化到MySQL
            sendAsyncUpdateMessage(cartSkuInfoDTO);
        } finally {
            redisLock.unlock(updateCartLockKey);
        }
        //返回购物车数据
        return queryCart(request.getUserId());
    }

    //更新购物车请求数量为0，删除缓存
    private void clearCartCache(CartSkuInfoDTO cartSkuInfoDTO) {
        Long userId = cartSkuInfoDTO.getUserId();
        String skuId = cartSkuInfoDTO.getSkuId();
        redisCache.hDel(RedisKeyConstant.SHOPPING_CART_HASH + userId, skuId);
        redisCache.hDel(RedisKeyConstant.SHOPPING_CART_EXTRA_HASH + userId, skuId);
        redisCache.zremove(RedisKeyConstant.SHOPPING_CART_ZSET + userId, skuId);
    }

    //更新购物车Redis缓存
    private void updateCartCache(CartSkuInfoDTO cartSkuInfoDTO) {
        //更新用户购物车的SKU数量缓存
        updateCartNumCache(cartSkuInfoDTO);
        //更新加入到购物车的SKU扩展信息缓存
        updateCartExtraCache(cartSkuInfoDTO);
        //更新用户购物车的SKU加购时间排序缓存
        updateCartSortCache(cartSkuInfoDTO);
    }
    
    //获取购物车中的商品
    private CartSkuInfoDTO getCartSkuInfoDTO(UpdateCookBookCartRequest request) {
        CartSkuInfoDTO cartSkuInfoDTO = cookbookCartConverter.requestToDTO(request);

        //获取购物车中的商品
        Object extra = redisCache.hGet(RedisKeyConstant.SHOPPING_CART_EXTRA_HASH + cartSkuInfoDTO.getUserId(), cartSkuInfoDTO.getSkuId());
        //购物车没有这个商品
        if (Objects.isNull(extra)) {
            throw new CookbookCartBizException(CookbookCartErrorCodeEnum.SKU_NOT_EXIST_CART_ERROR, CookbookCartErrorCodeEnum.SKU_NOT_EXIST_CART_ERROR.getErrorCode());
        }

        String cartSkuInfoString = extra.toString();
        CartSkuInfoDTO oldCartSkuInfoDTO = JsonUtil.json2Object(cartSkuInfoString, CartSkuInfoDTO.class);
        //购物车没有这个商品
        if (Objects.isNull(oldCartSkuInfoDTO)) {
            throw new CookbookCartBizException(CookbookCartErrorCodeEnum.SKU_NOT_EXIST_CART_ERROR, CookbookCartErrorCodeEnum.SKU_NOT_EXIST_CART_ERROR.getErrorCode());
        }

        cartSkuInfoDTO.setCount(request.getCount());
        return cartSkuInfoDTO;
    }
}
```

 


**7\.购物车的选中提交功能**


也要使用分布式锁保证请求的幂等性。



```
@RestController
@RequestMapping("/api/goodscart")
public class CookBookCartController {
    ...
    //选中购物车中的商品SKU项进行提交确认订单
    @RequestMapping("/checkedCartGoods")
    public JsonResult checkedCartGoods(@RequestBody @Valid CheckedCartRequest request){
        CookBookCartInfoDTO dto = cookBookCartService.checkedCartGoods(request);
        return JsonResult.buildSuccess(dto);
    }
    ...
}

@Service
public class CookBookCartServiceImpl implements CookBookCartService {
    ...
    @Transactional(rollbackFor = Exception.class)
    @Override
    public CookBookCartInfoDTO checkedCartGoods(CheckedCartRequest request) {
        String updateCartLockKey = RedisLockKeyConstants.UPDATE_CART_LOCK_KEY + request.getUserId() + ":" + request.getSkuId();
        boolean locked = redisLock.blockedLock(updateCartLockKey);

        if (!locked) {
            throw new BaseBizException("选中购物车失败");
        }

        try {
            //对象转换
            CartSkuInfoDTO cartSkuInfoDTO = cookbookCartConverter.requestToDTO(request);
            //更新加入到购物车的SKU扩展信息缓存
            updateCartExtraCache(cartSkuInfoDTO);
            //异步更新MySQL
            sendAsyncUpdateMessage(cartSkuInfoDTO);
        } finally {
            redisLock.unlock(updateCartLockKey);
        }

        //获取新的购物车数据
        return queryCart(request.getUserId());
    }
}
```

 


**8\.简单总结**


一.读多写多，如果使用Redis作主存储，先写Redis再写MySQL，那么如果写多个key，Redis写一半key，系统就宕机，可利用lua保证事务性。如果Redis写完，系统宕机，MySQL没写，此时只有缓存有数据。可以考虑先顺序写磁盘或者先写操作系统Page Cache，然后再写Redis缓存。这样即便系统宕机，也可以在重启的时候从文件中恢复数据。


 


二.读多写少，如果使用MySQL作主存储，先写MySQL再写Redis。那么只要数据写到MySQL，就一定可以同步到Redis。比如通过Canal监听MySQL的binlog，只要写入MySQL，Canal就可以监听到binlog并发送消息到MQ。


 


 本博客参考[豆荚加速器](https://yirou.org)。转载请注明出处！
