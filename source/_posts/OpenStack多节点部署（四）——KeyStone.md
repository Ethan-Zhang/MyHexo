title: OpenStack多节点部署（四）——KeyStone
date: 2013-02-04 15:12:10
tags: [OpenStack]
---
* [__OpenStack多节点部署（一）——服务器选型__](/2013/01/31/OpenStack多节点部署（一）——-服务器选型/)
* [__OpenStack多节点部署（二）——操作系统安装__](/2013/02/01/OpenStack多节点部署（二）——操作系统安装/)
* [__OpenStack多节点部署（三）——网络配置__](2013/02/04/OpenStack多节点部署（三）——网络配置/)
* [__OpenStack多节点部署（四）——KeyStone__](2013/02/04/OpenStack多节点部署（四）——KeyStone/)
* __OpenStack多节点部署（五）——Nova__
* __OpenStack多节点部署（六）——glance__

前面啰嗦了这么多，终于要正式进入OpenStack各组件安装部署的章节了。首先为大家带来的是OpenStack的用户登陆鉴权组件，KeyStone的安装。
首先，安装mysql服务，并分别创建Nova, glance, swift等组件独立的用户和口令.
```
sudo apt-get install mysql-server python-mysqldb
```
安装过程中提示设置密码，这里设置为mygreatsecret
```
sed -i '/bind-address/ s/127.0.0.1/0.0.0.0/' /etc/mysql/my.cnf
sudo restart mysql
sudo mysql -uroot -pmygreatsecret -e 'CREATE DATABASE nova;'
sudo mysql -uroot -pmygreatsecret -e 'CREATE USER novadbadmin;'
sudo mysql -uroot -pmygreatsecret -e "GRANT ALL PRIVILEGES ON nova.* TO 'novadbadmin'@'%';"
sudo mysql -uroot -pmygreatsecret -e "SET PASSWORD FOR 'novadbadmin'@'%' = PASSWORD('novasecret');"
sudo mysql -uroot -pmygreatsecret -e 'CREATE DATABASE glance;'
sudo mysql -uroot -pmygreatsecret -e 'CREATE USER glancedbadmin;'
sudo mysql -uroot -pmygreatsecret -e "GRANT ALL PRIVILEGES ON glance.* TO 'glancedbadmin'@'%';"
sudo mysql -uroot -pmygreatsecret -e "SET PASSWORD FOR 'glancedbadmin'@'%' = PASSWORD('glancesecret');"
sudo mysql -uroot -pmygreatsecret -e 'CREATE DATABASE keystone;'
sudo mysql -uroot -pmygreatsecret -e 'CREATE USER keystonedbadmin;'
sudo mysql -uroot -pmygreatsecret -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystonedbadmin'@'%';"
sudo mysql -uroot -pmygreatsecret -e "SET PASSWORD FOR 'keystonedbadmin'@'%' = PASSWORD('keystonesecret');"
```
安装KeyStone组件
```
sudo apt-get install keystone python-keystone python-keystoneclient
sed -i '/admin_token/ s/ADMIN/admin/' /etc/keystone/keystone.conf
sed -i '/connection/ s/sqlite\:\/\/\/\/var\/lib\/keystone\/keystone.db/mysql\:\/\/keystonedbadmin\:keystonesecret@192.168.3.1\/keystone/' /etc/keystone/keystone.conf
#注意修改mysql服务器地址
sudo service keystone restart
sudo keystone-manage db_sync
export SERVICE_ENDPOINT="http://localhost:35357/v2.0"
export SERVICE_TOKEN=admin
```
后面就是按照文档，创建租户Tenants，创建用户Users，创建角色Roles，最后进行租户、用户、角色之间的关联。不管创建什么类型，都会返回一个UID值，后面的步骤会用到前面的id，比如用户角色关联命令
```
keystone user-role-add --user $USER_ID --role $ROLE_ID --tenant_id $TENANT_ID
```
这个$USER_ID和$ROLE_ID等就是前面创建用户或者角色时候得到的ID
比如先创建用户
```
keystone user-create --name admin --pass admin --email admin@foobar.com
```
查看ID
```
keystone user-list
+----------------------------------+---------+-------------------+--------+
|                id                | enabled |       email       |  name  |
+----------------------------------+---------+-------------------+--------+
| b3de3aeec2544f0f90b9cbfe8b8b7acd | True    | admin@foobar.com  | admin  |
| ce8cd56ca8824f5d845ba6ed015e9494 | True    | nova@foobar.com   | nova   |
+----------------------------------+---------+-------------------+--------+
```
如上，我们创建的名字为admin的用户就会显示出来，后面的步骤就要用这个ID。
大家会发现这样最非常麻烦，而且id这样拷贝很容易出错，所以我们要用脚本来自动完成上面的这些操作，以及service endpoint的操作。
```Bash
#!/bin/bash
#
# Initial data for Keystone using python-keystoneclient
#
# Tenant               User      Roles
# ------------------------------------------------------------------
# admin                admin     admin
# service              glance    admin
# service              nova      admin, [ResellerAdmin (swift only)]
# service              quantum   admin        # if enabled
# service              swift     admin        # if enabled
# service              cinder    admin        # if enabled
# service              heat      admin        # if enabled
# demo                 admin     admin
# demo                 demo      Member, anotherrole
# invisible_to_admin   demo      Member
# Tempest Only:
# alt_demo             alt_demo  Member
#
# Variables set before calling this script:
# SERVICE_TOKEN - aka admin_token in keystone.conf
# SERVICE_ENDPOINT - local Keystone admin endpoint
# SERVICE_TENANT_NAME - name of tenant containing service accounts
# SERVICE_HOST - host used for endpoint creation
# ENABLED_SERVICES - stack.sh's list of services to start
# DEVSTACK_DIR - Top-level DevStack directory
# KEYSTONE_CATALOG_BACKEND - used to determine service catalog creation
SERVICE_HOST=${SERVICE_HOST:-192.168.3.1}
#将这个IP修改为Keystone服务器的内网IP
SERVICE_TOKEN=${SERVICE_TOKEN:-admin}
SERVICE_ENDPOINT=${SERVICE_ENDPOINT:-http://localhost:35357/v2.0}
# Defaults
export SERVICE_TOKEN=$SERVICE_TOKEN
export SERVICE_ENDPOINT=$SERVICE_ENDPOINT
SERVICE_TENANT_NAME=${SERVICE_TENANT_NAME:-service}

function get_id () {
    echo `"$@" | awk '/ id / { print $4 }'`  #  '$@'代表函数的参数，参数就是get_id后面接的KeyStone命令
}


# Tenants
# -------

ADMIN_TENANT=$(get_id keystone tenant-create --name=admin)
SERVICE_TENANT=$(get_id keystone tenant-create --name=$SERVICE_TENANT_NAME)


# Users
# -----

ADMIN_USER=$(get_id keystone user-create --name=admin \
                                         --pass=admin \
                                         --email=admin@example.com)
NOVA_USER=$(get_id keystone user-create --name=nova \
                                        --pass=nova \
                                        --email=demo@example.com)
GLANCE_USER=$(get_id keystone user-create --name=glance \
                                        --pass=glance \
                                        --email=demo@example.com)
SWIFT_USER=$(get_id keystone user-create --name=swift \
                                        --pass=swift \
                                      --email=demo@example.com)
# Roles
# -----

ADMIN_ROLE=$(get_id keystone role-create --name=admin)
# ANOTHER_ROLE demonstrates that an arbitrary role may be created and used
# TODO(sleepsonthefloor): show how this can be used for rbac in the future!
MEMBER_ROLE=$(get_id keystone role-create --name=Member)


# Add Roles to Users in Tenants
keystone user-role-add --user_id $ADMIN_USER --role_id $ADMIN_ROLE --tenant_id $ADMIN_TENANT
keystone user-role-add --user_id $NOVA_USER --role_id $ADMIN_ROLE --tenant_id $SERVICE_TENANT
keystone user-role-add --user_id $GLANCE_USER --role_id $ADMIN_ROLE --tenant_id $SERVICE_TENANT
keystone user-role-add --user_id $SWIFT_USER --role_id $ADMIN_ROLE --tenant_id $SERVICE_TENANT

# The Member role is used by Horizon and Swift so we need to keep it:
keystone user-role-add --user_id $ADMIN_USER --role_id $MEMBER_ROLE --tenant_id $ADMIN_TENANT


# Services
# --------

# Keystone

	KEYSTONE_SERVICE=$(get_id keystone service-create \
		--name=keystone \
		--type=identity \
		--description="Keystone Identity Service")
	keystone endpoint-create \
	    --region RegionOne \
		--service_id $KEYSTONE_SERVICE \
		--publicurl "http://$SERVICE_HOST:5000/v2.0" \
		--adminurl "http://$SERVICE_HOST:35357/v2.0" \
		--internalurl "http://$SERVICE_HOST:5000/v2.0"


# Nova
        NOVA_SERVICE=$(get_id keystone service-create \
            --name=nova \
            --type=compute \
            --description="Nova Compute Service")
        keystone endpoint-create \
            --region RegionOne \
            --service_id $NOVA_SERVICE \
            --publicurl "http://$SERVICE_HOST:8774/v2/\$(tenant_id)s" \
            --adminurl "http://$SERVICE_HOST:8774/v2/\$(tenant_id)s" \
            --internalurl "http://$SERVICE_HOST:8774/v2/\$(tenant_id)s"

    # Nova needs ResellerAdmin role to download images when accessing
    # swift through the s3 api. The admin role in swift allows a user
    # to act as an admin for their tenant, but ResellerAdmin is needed
    # for a user to act as any tenant. The name of this role is also
    # configurable in swift-proxy.conf
    #RESELLER_ROLE=$(get_id keystone role-create --name=ResellerAdmin)
    #keystone user-role-add \
    #    --tenant_id $SERVICE_TENANT \
    #    --user_id $NOVA_USER \
    #    --role_id $RESELLER_ROLE


# Volume
        VOLUME_SERVICE=$(get_id keystone service-create \
            --name=volume \
            --type=volume \
            --description="Volume Service")
        keystone endpoint-create \
            --region RegionOne \
            --service_id $VOLUME_SERVICE \
            --publicurl "http://$SERVICE_HOST:8776/v1/\$(tenant_id)s" \
            --adminurl "http://$SERVICE_HOST:8776/v1/\$(tenant_id)s" \
            --internalurl "http://$SERVICE_HOST:8776/v1/\$(tenant_id)s"




# Glance
        GLANCE_SERVICE=$(get_id keystone service-create \
            --name=glance \
            --type=image \
            --description="Glance Image Service")
        keystone endpoint-create \
            --region RegionOne \
            --service_id $GLANCE_SERVICE \
            --publicurl "http://$SERVICE_HOST:9292/v1" \
            --adminurl "http://$SERVICE_HOST:9292/v1" \
            --internalurl "http://$SERVICE_HOST:9292/v1"


# Swift
        SWIFT_SERVICE=$(get_id keystone service-create \
            --name=swift \
            --type="object-store" \
            --description="Swift Service")
        keystone endpoint-create \
            --region RegionOne \
            --service_id $SWIFT_SERVICE \
            --publicurl "http://$SERVICE_HOST:8080/v1/AUTH_\$(tenant_id)s" \
            --adminurl "http://$SERVICE_HOST:8080/v1" \
            --internalurl "http://$SERVICE_HOST:8080/v1/AUTH_\$(tenant_id)s"



# EC2
        EC2_SERVICE=$(get_id keystone service-create \
            --name=ec2 \
            --type=ec2 \
            --description="EC2 Compatibility Layer")
        keystone endpoint-create \
            --region RegionOne \
            --service_id $EC2_SERVICE \
            --publicurl "http://$SERVICE_HOST:8773/services/Cloud" \
            --adminurl "http://$SERVICE_HOST:8773/services/Admin" \
            --internalurl "http://$SERVICE_HOST:8773/services/Cloud"
```
最后，用命令验证查看KeyStone是否安装正确
```
keystone tenant-list
keystone user-list
keystone role-list
keystone service-list
```
好了，有关KeyStone的相关部署方法就介绍到这里。
