# HUE前端搭建

**基础环境**

```text
centos7.6 1core2g
```



## 1.MySQL搭建

```shell
# 1.官网下载Yum Repo
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm

# 2.yum安装
yum -y install mysql57-community-release-el7-10.noarch.rpm

# 3.安装MySQL服务器
yum -y install mysql-community-server

# 4.启动MySQL数据库服务
systemctl start mysql

# 查看初始设置root密码
grep "password" /var/log/mysqld.log
# 使用初始密码进入musql中
mysql -uroot -p
输入密码

# 更改初始密码
出现报错：
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

因为：
原来MySQL5.6.6版本之后增加了密码强度验证插件validate_password，相关参数设置的较为严格。
使用了该插件会检查设置的密码是否符合当前设置的强度规则，若不满足则拒绝设置。

# 解决方案
mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_dictionary_file    |        |
| validate_password_length             | 8      |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+

参数解释：
validate_password_dictionary_file
插件用于验证密码强度的字典文件路径。

validate_password_length
密码最小长度，参数默认为8，它有最小值的限制，最小值为：validate_password_number_count + validate_password_special_char_count + (2 * validate_password_mixed_case_count)

validate_password_mixed_case_count
密码至少要包含的小写字母个数和大写字母个数。

validate_password_number_count
密码至少要包含的数字个数。

validate_password_policy
密码强度检查等级，0/LOW、1/MEDIUM、2/STRONG。

修改MySQL参数配置
mysql> set global validate_password_mixed_case_count=0;
Query OK, 0 rows affected (0.00 sec)
 
mysql> set global validate_password_number_count=6;
Query OK, 0 rows affected (0.00 sec)
 
mysql> set global validate_password_special_char_count=0;
Query OK, 0 rows affected (0.00 sec)
 
mysql> set global validate_password_length=6;
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| validate_password_check_user_name    | OFF   |
| validate_password_dictionary_file    |       |
| validate_password_length             | 6     |
| validate_password_mixed_case_count   | 0     |
| validate_password_number_count       | 6     |
| validate_password_policy             | LOW   |
| validate_password_special_char_count | 0     |
+--------------------------------------+-------+

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
这样就解决了
```



