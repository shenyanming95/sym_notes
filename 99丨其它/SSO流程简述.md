跨域的单点登录，一般使用CAS，它的标准流程如下：

1. 用户访问app系统，app系统需要登录，用户此时没有登录；
2. 跳转到CAS server，即SSO登录系统；SSO系统此时也没有登录，弹出用户登录页；
3. 用户填写用户名/密码，SSO系统认证后，将登录状态写入SSO的session，浏览器写入SSO系统域下的cookie；
4. SSO系统登录完成后生成一个ST（service ticket）跳转到app系统，同时将ST作为参数传递给app系统；
5. app系统拿到ST后，从后台向SSO系统发送请求，验证ST是否有效；
6. 验证通过后，app系统将登录状态写入session并设置app系统域下的cookie；

至此，用户跨域单点登录就完成，后续访问app系统，就是登录；如果此时要登录app2系统：

1. 用户访问app2系统，app2系统没有登录，跳转到SSO系统；
2. SSO系统此时已经登录了，不需要重新登录认证；
3. SSO生成ST，浏览器跳转到app2系统，并将ST作为参数传递给app2系统
4. app2拿到ST，后台访问SSO系统，验证ST是否有效；
5. 验证成功后，app2系统将登录状态写入session，并在app2系统域下写入cookie；

这样，app2系统不需要走登录流程，就已经是登录了。SSO，app和app2在不同的域，它们之间的session不共享也是没问题的。