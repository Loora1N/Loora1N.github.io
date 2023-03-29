---
title: 【CTFd 二次开发】解题播报机器人 V2
index_img: img/article/botv2/image-20221108200108487.png
excerpt: 实现CTFd创建新题目的播报以及题目信息更新时，进行播报
---
> 本文基于 CTFd Version 3.4.3 开发，可能存在源码不相同的情况，阅读时请注意文章时效性。

## 更新内容

- 实现CTFd创建新题目的播报，且仅在题目状态由`hidden`转为`visible`时播报
- 实现题目信息更新时，进行播报

![效果展示](img/article/botv2/image-20221108200108487.png)

## 前端页面修改

### appearance.html

打开`CTFd/CTFd/themes/admin/templates/configs/appearance.html`，在上次更新的基础上增加题目创建播报文案，和题目信息更新播报文案接口。这里直接放出完整html，可自行对照

```html
<div role="tabpanel" class="tab-pane config-section active" id="appearance">
	<form method="POST" autocomplete="off" class="w-100">
		<div class="form-group">
			<label for="ctf_name">
				Competition Name
				<small class="form-text text-muted">Competition name displayed instead of a logo</small>
			</label>
			<input class="form-control" id='ctf_name' name='ctf_name' type='text' placeholder="CTF Name"
				   {% if ctf_name is defined and ctf_name != None %}value="{{ ctf_name }}"{% endif %}>
		</div>

		<div class="form-group">
			<label>
				CTF Description<br>
				<small class="form-text text-muted">
					Description for the CTF
				</small>
			</label>
			<textarea class="form-control" type="text" id="ctf_description" name="ctf_description" rows="5">{{ ctf_description }}</textarea>
		</div>
        <div class="form-group">
            <label for="bot-ip">
                Bot Address<br>
                <small class="form-text text-muted">
                    the address of QQ-bot
                </small>
                <small class="form-text text-muted">
                    eg. 127.0.0.1:1234
                </small>
            </label>
            <input class="form-control" id='bot-ip' name='bot_ip' type='text'
					{% if bot_ip is defined and bot_ip != None %}value="{{ bot_ip }}"{% endif %}>
        </div>

        <div class="form-group">
            <label for="group-id">
                QQ Group Number<br>
            </label>
            <input class="form-control" id='group-id' name='group_id' type='text'
					{% if group_id is defined and group_id != None %}value="{{ group_id }}"{% endif %}>
        </div>


        <div class="form-group">
            <label for="bot-text">
                解题播报消息格式<br>
                <small class="form-text text-muted">
                    ep. 恭喜%s做出题目%s，默认第一个参数为用户名，第二个参数为题目名称
                </small>
            </label>
            <input class="form-control" id='bot-text' name='bottext' type='text'
				   {% if bottext is defined and bottext != None %}value="{{ bottext }}"{% endif %}>
        </div>

        <div class="form-group">
            <label for="create-text">
                题目可见播报<br>
                <small class="form-text text-muted">
                    ep. 题目%s已登入平台
                </small>
            </label>
            <input class="form-control" id='create-text' name='createtext' type='text'
				   {% if createtext is defined and createtext != None %}value="{{ createtext }}"{% endif %}>
        </div>

        <div class="form-group">
            <label for="update-text">
                题目更新播报<br>
                <small class="form-text text-muted">
                    ep. 题目%s内容已更新
                </small>
            </label>
            <input class="form-control" id='update-text' name='updatetext' type='text'
				   {% if updatetext is defined and updatetext != None %}value="{{ updatetext }}"{% endif %}>
        </div>

        <div class="form-group">
            <label for="botof">
                Turn on the Bot
            </label>
            <div>
                <select class="form-control custom-select" id="botof" name="bot">
                    <option value=0 {% if bot == 0 %}selected{% endif %}>
                        OFF
                    </option>
                    <option value=1 {% if bot == 1 %}selected{% endif %}>
                        ON
                    </option>
                </select>
            </div>
        </div>

		<button type="submit" class="btn btn-md btn-primary float-right">Update</button>
	</form>
</div>
```

### 效果图

![image-20221108200540771](img/article/botv2/image-20221108200540771.png)

## 后端更改

### views.py

打开`CTFd/CTFd/views.py`，添加变量`createtext`和`updatetext`，用于接收html页面数据

![image-20221108200717361](img/article/botv2/image-20221108200717361.png)

```python
# Robot
bot = request.form.get("bot")
bottext = request.form.get("bottext")
createtext = request.form.get("createtext")
updatetext = updatetext.form.get("updatetext")
group_id = request.form.get("group_id") #qq群号
bot_ip = request.form.get("bot_ip") #机器人服务地址,如 127.0.0.1:8000
set_config("bot", bot)
set_config("bottext", bottext)
set_config("createtext", createtext)
set_config("updatetext", updatetext)
set_config("group_id",group_id)
set_config("bot_ip",bot_ip)
```

### challenge.py

打开`CTFd/CTFD/api/v1/challenges.py`，在`patch`函数内增加一段代码，在更新题目信息后进行逻辑判断，后调用机器人的API

![image-20221108201009252](img/article/botv2/image-20221108201009252.png)

完整函数代码如下：

```python
def patch(self, challenge_id):
    data = request.get_json()
    #TODO:
    # Load data through schema for validation but not for insertion
    schema = ChallengeSchema()
    response = schema.load(data)
    if response.errors:
        return {"success": False, "errors": response.errors}, 400

    challenge = Challenges.query.filter_by(id=challenge_id).first_or_404()
    state_before = challenge.state
    challenge_class = get_chal_class(challenge.type)
    challenge = challenge_class.update(challenge, request)
    response = challenge_class.read(challenge)
    #ROBOT send message when update the information of challenge
    bot = get_config("bot")
    if (bot):
        group_id = get_config("group_id")
        bot_ip = get_config("bot_ip")
        boturl = "http://"+str(bot_ip)+"/send_group_msg?group_id="+str(group_id)+"&message="
        if(challenge.state!="hidden" and challenge.state!="locked"):
            if(state_before=="hidden"):
                bottext = get_config("createtext")
                botmessage = (bottext) % (challenge.name)
                url = boturl + botmessage
                requests.get(str(url))
             elif(state_before!="hidden"):
                bottext = get_config("updatetext")
                botmessage = (bottext) % (challenge.name)
                url = boturl + botmessage
                requests.get(str(url))

    return {"success": True, "data": response}
```

>后续在增加一二三血吧，这个功能略有点麻烦