# 玩家鉴权

## 1、什么是玩家鉴权

玩家鉴权是验证玩家访问**TGOS**服务权利的过程。**TGOS**本身不提供实质的账号服务系统，通过对接第三方账号服务系统（例如**Tencent IntlSDK**），在后端验证账号票据确认玩家账号有效性，进而让玩家客户端获得访问**TGOS**服务的权利。

## 2、关键词释义

- Account Service: 账号服务系统（例如 **Tencent IntlSDK**）
- Tencent IntlSDK：腾讯为海外游戏提供的发行期技术解决方案，包括账号登录、事件上报等功能
- open_id: **Account Service** 提供的用户标识
- token: **open_id** 对应的有效票据，可能需要通过 **Account Service** 续票
- player_id： **TGOS** 体系提供的玩家标识，全局唯一
- player_ticket: 玩家鉴权有效票据，**TGOS** 自动续票（**SDK** 已屏蔽，开发者无需关心）
- player_session: 从玩家 **TGOS** 鉴权成功，到玩家退出或者鉴权失效的这一个会话过程。一旦玩家会话失效，游戏客户端将不能从 **TGOS** 获得服务。
- player_session data: 与 **player_session** 关联的一组公开的 **kv-data**，可用于存储玩家的在线状态，网络等，这组数据是非持久的，玩家会话失效后，数据被移除。（后期实现）
- fake account service: 模拟 **Account Service**，帮助开发商在未接入实际的 **Account Service** 前， 跑通所有依赖发行账号服务的 **TGOS** 功能。
- account_chn: 账号渠道，**0: fake account service, 1: Tencent IntlSDK**

## 3、 鉴权时序图

```sequence

Game Client->Account Service: login
Account Service-->>Game Client: return: open_id & token
Game Client->TGOS SDK: StartPlayerSession(open_id, token)
TGOS SDK-->>Game Client: success: player_id, faied: error msg
TGOS SDK->>TGOS Backend: StartPlayerSession(open_id, token, account_chn)
TGOS Backend->Account Service:Auth(open_id, token)
TGOS Backend-->>Account Service: success:player_id & player_ticket
TGOS Backend-->>TGOS SDK: success:player_id & player_ticket
```

## 4、 关键接口说明

- player session 事件监听

```c

/*
*	@brief	set callback for player session event.
*	@param	callback	result callback for the function
*	@return
*/
void SetEventCallback(std::function<void(PlayerSessionEvt evt,
const std::string &msg)> callback)
// event define for player session enum class PlayerSessionEvt : int {
StartSuccess = 0,
SessionExpired_HeartbeatFailed = 1,
SessionExpired_TicketExpired = 2,
SessionExpired_RestartFailed = 3,
};

```

- 鉴权登录

  - open_id 和 token 可以在登录成功后从 Account Service 返回信息得到；
  - title_region_id 可以从选区返回的信息得到；在 web portal 建立区服时可确认 title_region_id

  ```c
    /*
    *	@brief	make an authorization for the logined account.
    *	@param	open_id logined account's open_id
    *	@param	token	logined account's token
    *	@param	title_region_id	region the player wants to login
    *	@param	callback	result callback for the function
    *	@return
    */
    void StartPlayerSession(const std::string& open_id,
    const std::string& token,
    const std::string& title_region_id, HttpCallback<http_model::AuthenticationInfo>
    callback);

    struct AuthenticationInfo {
    bool is_first_login; // is this account playing the game for the first time
    std::string player_id; // palyer id corresponding to the logined account std::string session_id;	// player's login session id
    };
  ```

- 刷新账号票据
  当 Account Service 的账号票据有更新时，要通过下面的接口通知 TGOS。

  ```c
  /*
  *	@brief	inform TGOS of the latest account token.
  *	@param	open_id logined account's open_id
  *	@param	token	logined account's token
  *	@param	callback	result callback for the function
  *	@return
  */
  void UpdateAccountToken(const std::string& open_id,
  const std::string& token, HttpEmptyResponseCallback callback)

  ```

- 玩家会话重连

  ```c
  /*
  *	@brief	try to make the player session be valid again.
  *	@note	get function result from player session event.
  *	@return
  */
  void RestartPlayerSession();

  ```

- 退出登录

  ```c
  /*
  *	@brief	close player access to TGOS.
  *	@param	callback	result callback for the function
  *	@return
  */
  void EndPlayerSession(HttpEmptyResponseCallback callback);

  ``
  ```

## 5、使用说明

- 可以通过下面的接口获得 IPlayerAuth 实例：

  ```c
  IPlayerAuth *PlayerAuth();
  ```

- 监听玩家会话事件

  ```c
  void SetEventCallback(std::function<void(PlayerSessionEvt evt,
  const std::string &msg)> callback);

  ```

- 当游戏客户端通过 Account Service（例如 Tencent IntlSDK）完成账号登录后，即可启动玩家会话：

  ```c
  void StartPlayerSession(const std::string& open_id,
  const std::string& token,
  const std::string& title_region_id, HttpCallback<http_model::AuthenticationInfo>
  callback);

  ```

- 当账号的 token 被更新后（与 Account Service 有关），要通过下面的接口把新 token 通知 TGOS

  ```c
  void UpdateAccountToken(const std::string& open_id,
  const std::string& token, HttpEmptyResponseCallback callback) ;

  ```

- 如果玩家会话失效（原因见 PlayerSessionEvt 的枚举），开发者可以考虑尝试重新使玩家会话有效，当然也可以放弃（TGOS 内部其实已经做了限时限次的重试）。一旦玩家会话失效，游戏客户端将不能从 TGOS 获得服务。

  ```c
  void RestartPlayerSession();
  ```

- 当玩家要结束玩家会话，需要调用下面 IPlayerAuth 的接口：

  ```c
  void EndPlayerSession(HttpEmptyResponseCallback callback);
  ```

- 如果玩家要切换账号，请使用下面的流程完成切换：
  - 调用 IPlayerAuth::EndPlayerSession()接口结束之前的玩家会话；
  - 通过 Account Service 切换账号并登录成功；
  - 调用 IPlayerAuth::StartPlayerSession()启动新的玩家会话；

## 6、 Fake Account System

模拟发行账号登录的鉴权服务，提供相关的接口和流程，帮助开发商在未接入实际的发行账号服务前， 跑通所有依赖发行账号服务的外部功能，它是发行账号服务的极简化功能版本的替代品。
可以通过下面的接口获得 IFakeAccountSystem 实例：

```c
IFakeAccountSystem* FakeAccountSystem();
```

它主要通过两个接口完成账号相关服务：

- 账号登录

  ```c
  /*
  *	@brief	account logins.
  *	@param	callback	result callback for the function
  *	@return
  */
  void Login(OnLoginResult callback);

  ```

- 刷新票据

  ```c
  /*
  *	@brief	refresh account token before token is expired.
  *	@param	token	last token
  *	@param	callback	result callback for the function
  *	@return
  */
  void RefreshToken(const std::string &token, OnRefreshTokenResult callback)

  ```
