# gitlab notes


## 修改密码

gitlab-rails console

user = User.find_by_username('root')
user.password = 'NewSecurePassword'
user.password_confirmation = 'NewSecurePassword'
user.save!

## external url

影响到点击克隆显示的地址
打开 GitLab 的主配置文件：/etc/gitlab/gitlab.rb。
修改
external_url 'http://your.gitlab.domain'

重新配置 gitlab 并重启
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart

3. 检查 SSH 和 HTTP/HTTPS 配置
gitlab_rails['gitlab_shell_ssh_host'] = 'your.gitlab.domain'

如果依然显示错误可尝试清理缓存

sudo gitlab-rake cache:clear

检查服务状态
sudo gitlab-ctl status

nginx 配置冲突
	•	确保没有其他服务占用 GitLab 的端口。
	•	检查 Nginx 配置是否生成正确：
sudo gitlab-ctl show-config | grep nginx