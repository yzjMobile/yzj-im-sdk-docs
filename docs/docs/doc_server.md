# 服务端集成文档




## 服务端基本架构

云之家IM服务端提供了服务端集成的demo工程im-server，包含了用户集成、管理、接口验证等基本功能，本文示例所用代码片段都来源于im-server工程。

企业集成云之家IM服务端后，会创建一个应用ID(appId)和对应的应用密钥（secret），每个appId对应的用户数据互相隔离。

appId与secret可用于生成接口请求的凭证（AccessKey），服务端通过AccessKey验证请求的合法性。

生成凭证示例代码：

    String accessKey = DigestUtils.sha1Hex(StringUtils.join(appId, secret, uid, timestamp));// uid 为空，则使用username



用户集成
--------

### 用户属性

| 字段            | 名称       | 说明                           |
|-----------------|------------|--------------------------------|
| id:string       | 用户ID     | PK，注册新用户时，系统自动生成 |
| username:string | 用户帐号   | UK                             |
| password:string | 用户密码   | 【可选】                       |
| appId:string    | 应用ID     |                                |
| phone:string    | 手机号码   |                                |
| email:string    | 邮箱地址   |                                |
| nickname:string | 昵称/名称  |                                |
| avatar:string   | 头像URL    |                                |
| gender:integer  | 性别       |                                |
| extUid:string   | 绑定用户ID | 例如万科用户ID              |


### 用户绑定

企业集成IM服务端的时候，需要把企业现有用户和新注册的用户与IM服务中的用户集成，为每个用户创建一个用户id(id),并与应用ID（appId）和企业现有用户id（extUid）绑定。
服务端代码集成

im-server使用Motan通过rpc的方式实现用户的集成与管理，UserService和UserAuthService提供具体的方法实现

Motan 相关配置

引入依赖
    
    "com.weibo:motan-springsupport:0.3.0",
    "com.weibo:motan-registry-zookeeper:0.3.0",
    "com.weibo:motan-transport-netty:0.3.0",
    "com.weibo:motan-core:0.3.0",
    
配置文件加入
    
    # Motan Aonnotation config
    motan.annotation.package=com.kingdee.imserver.rest
    
    # Motan Registry config
    motan.registry.regProtocol=zookeeper
    motan.registry.address=172.20.181.39:2181 // 实际motan注册地址及端口
    motan.registry.connectTimeout=2000
    motan.registry.requestTimeout=5000
    
    # Motan Protocol config
    motan.protocol.name=motan
    motan.protocol.defaultConfig=true

代码注入依赖

    @MotanReferer
    private UserService userService;

    @MotanReferer
    private UserAuthService userAuthService;


UserAuthService	提供方法如下

- 注册新用户
- 修改用户数据
- 设置用户密码
- 获取用户数据
- 获取用户列表
- 分页获取所有用户
- 删除(禁用)用户
- 启用(激活)

	
UserService 提供方法如下
	
- 帐号登录
- 根据帐号登出
- 根据token登出


代码集成示例
--------

### 注册新用户 
此方法同时将用户id、appId、extUid绑定。

具体代码路径： com.kingdee.imserver.rest.User.java

    @MotanReferer
    private UserService userService;

    @RequestMapping(value = "/create")
	public Result create(@RequestBody UserCreateRequest request) {

        UserCreateByAppIdDTO dto = new UserCreateByAppIdDTO();
        dto.setUsername(request.getUsername());
		dto.setAppId(request.getAppId());
		...
        dto.setExtUid(request.getExtUid());
        dto.setAccessTime(System.currentTimeMillis());
        String accessKeyInput = DigestUtils.shaHex(StringUtils.join(dto.getAppId(), theAppSecret, dto.getUsername(), String.valueOf(dto.getAccessTime())));
        dto.setAccessKey(accessKeyInput);
		// 通过rpc的方式调用创建用户的方法，
        RpcResult<UserVO> result = userService.create(dto);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        return Result.build(result.getData());
    }



### 修改用户数据 

    @RequestMapping(value = "/update")
    public Result update(@RequestBody UserUpdateRequest request) {

        UserUpdateByAppIdDTO dto = new UserUpdateByAppIdDTO();
        dto.setId(request.getUid());
        dto.setUsername(request.getUsername());
        ...
        dto.setExtUid(request.getExtUid());
        dto.setAccessTime(System.currentTimeMillis());
        String value = StringUtils.isNotEmpty(dto.getId()) ? dto.getId() : (dto.getUsername() == null ? "" : dto.getUsername());
        String accessKeyInput = DigestUtils.shaHex(StringUtils.join(dto.getAppId(), theAppSecret, value, String.valueOf(dto.getAccessTime())));
        dto.setAccessKey(accessKeyInput);
        // 通过rpc的方式调用创建用户的api
        RpcResult<UserVO> result = userService.update(dto);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        return Result.build(result.getData());
    }

### 设置用户密码 

    @RequestMapping(value = "/setPassword")
    public Result setPassword(@RequestBody UserSetPasswordRequest request) {
        UserUpdatePwdDTO userUpdatePwdDTO = new UserUpdatePwdDTO();
        userUpdatePwdDTO.setId(request.getUid());
        userUpdatePwdDTO.setUsername(request.getUsername());
        userUpdatePwdDTO.setNewPassword(request.getNewPassword());
        RpcResult<Boolean> result = userService.updatePassword(userUpdatePwdDTO);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        return Result.EMPTY;
    }

### 获取用户数据

    @RequestMapping(value = "/get")
    public Result get(@RequestBody UserGetRequest request) {

        UserGetDTO userGetDTO = new UserGetDTO();
        userGetDTO.setId(request.getUid());
        userGetDTO.setUsername(request.getUsername());
        RpcResult<UserVO> result = userService.get(userGetDTO);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        return Result.build(result.getData());
    }

### 获取用户列表

    @RequestMapping(value = "/list")
    public Result list(@RequestBody UserListRequest request) {

        List<String> uids = request.getUids();
        if (uids == null || uids.isEmpty()) {
            return Result.EMPTY;
        }
        Set<String> set = new HashSet<>(uids);
        RpcResult<List<UserVO>> result = userService.list(set);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        return Result.build(result.getData());
    }

### 分页获取所有用户

    @RequestMapping(value = "/all")
    public Result all(@RequestBody UserAllRequest request) {
        Map<String, Object> filters = new HashMap<String, Object>();
        filters.put("appId", theAppId);
        Pageable pageable = new PageRequest(request.getPage(),request.getSize(), Sort.Direction.DESC,"id");
        RpcResult<Page<UserVO>> result = userService.search(filters, pageable);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        return Result.build(result.getData());
    }

### 删除(禁用)用户

    @RequestMapping(value = "/remove")
    public Result remove(@RequestBody UserRemoveRequest request) {

        List<String> uids = request.getUids();
        if (uids == null || uids.isEmpty()) {
            return Result.EMPTY;
        }
        Set<String> set = new HashSet<>(uids);
        RpcResult<Boolean> result = userService.remove(set);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        return Result.EMPTY;
    }

### 启用(激活)

    @RequestMapping(value = "/activate")
        public Result activate(@RequestBody UserActivateRequest request) {
    
            List<String> uids = request.getUids();
            if (uids == null || uids.isEmpty()) {
                return Result.EMPTY;
            }
            Set<String> set = new HashSet<>(uids);
            RpcResult<Boolean> result = userService.activate(set);
            if (!result.isSuccess()) {
                return Result.build(result.getErrorCode(), result.getError());
            }
    
            return Result.EMPTY;
        }
    

### 用户登录

代码路径：com.kingdee.imserver.rest.Auth.java


	public Result login(@RequestBody UserLoginRequest request) {
        ...
        
        UserLoginByAppId dto = new UserLoginByAppId();
        dto.setUid(request.getUid());
        dto.setUsername(request.getUsername());
        dto.setAccessTime(System.currentTimeMillis());
        dto.setAppId(theAppId);
        String value = StringUtils.isNotEmpty(dto.getUid()) ? dto.getUid() : (dto.getUsername() == null ? "" : dto.getUsername());
        String accessKeyInput = DigestUtils.sha1Hex(StringUtils.join(dto.getAppId(), theAppSecret, value, String.valueOf(dto.getAccessTime())));
        dto.setAccessKey(accessKeyInput);
		// 通过rpc的方式调用登录方法
        RpcResult<UserLoginResultVO> result = userAuthService.login(dto);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        return Result.build(result.getData());
    }
    
### 根据帐号登出

	@RequestMapping(value = "/logout")
        public Result logout(@RequestBody UserLogoutRequest request) {
            if (StringUtils.isNotEmpty(request.getUid())) {
                userAuthService.logoutByUid(request.getUid());
            } else {
                userAuthService.logoutByUsername(request.getUsername());
            }
    
            return Result.EMPTY;
        }
    
### 根据token登出

	@RequestMapping(value = "/logoutByToken")
    public Result logoutByToken(@RequestBody UserLogoutByTokenRequest request) {
        UserAuthValidateTokenDTO userAuthValidateTokenDTO = new UserAuthValidateTokenDTO();
        userAuthValidateTokenDTO.setToken(request.getToken());
        RpcResult<UserAuthVO> result = userAuthService.validateToken(userAuthValidateTokenDTO);
        if (!result.isSuccess()) {
            return Result.build(result.getErrorCode(), result.getError());
        }

        UserAuthVO userAuthVO = result.getData();
        userAuthService.logoutByUid(userAuthVO.getId());
        return Result.EMPTY;
    }
