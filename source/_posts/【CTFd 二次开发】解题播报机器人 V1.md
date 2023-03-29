---
title: 【CTFd 二次开发】解题播报机器人 V1
index_img: img/article/botv1/image-20221011093547622.png
excerpt: 本文基于 CTFd Version 3.4.3 开发，可能存在源码不相同的情况，阅读时请注意文章时效性。机器人会一直更新，后续应该会有一二三血播报等更详尽的功能
---

> 本文基于 CTFd Version 3.4.3 开发，可能存在源码不相同的情况，阅读时请注意文章时效性。机器人会一直更新，后续应该会有一二三血播报等更详尽的功能
>
> 注：本文不做机器人相关代码更改，直接利用nonebot自带接口实现

## 前端页面修改

### config.html

打开`CTFd/CTFd/themes/admin/templates/config.html`，更改左侧栏目标题

![config.html](img/article/botv1/image-20221011092121090.png)

### apperance.html

打开`CTFd/CTFd/themes/admin/templates/configs/appearance.html`，在其中插入代码，完整html如下

> 因可能存在CTFd不同版本页面不同的问题，请视具体情况更改html页面

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
                Message<br>
                <small class="form-text text-muted">
                    ep. 恭喜%s做出题目%s，默认第一个参数为用户名，第二个参数为题目名称
                </small>
            </label>
            <input class="form-control" id='bot-text' name='bottext' type='text'
				   {% if bottext is defined and bottext != None %}value="{{ bottext }}"{% endif %}>
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



### 效果展示

![前端效果](img/article/botv1/image-20221011093547622.png)

这样就可以在设置页面随时更改了， `Bot Address`是Bot接收http请求的地址，`QQ Group Number`是机器人所在的qq群号。如果机器人运行的地址发生变化，或想更改发消息的群号，只需在这里更改即可，目前还不支持同时多个群发送消息，准备等到添加一二三血功能时一起添加。



## 后端更改

### view.py

打开`CTFd/CTFd/views.py`，添加几个变量，用于接收html页面数据

![view.py](img/article/botv1/image-20221011092849492.png)



### challenges.py

打开`CTFd/CTFD/api/v1/challenges.py`，在解题成功后，调用机器人API即可

![challenges.py](img/article/botv1/image-20221011093425463.png)



### 运行演示

![提交flag](img/article/botv1/image-20221011094014448.png)

![群消息成功发送](img/article/botv1/image-20221011094033270.png)