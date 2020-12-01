# ClouderaApacheHueUserAutomation
### Automated creation/deactivation of users in Cloudera/Apache Hue

Here will be described Cloudera release of Hue (CM/CDH 6.1.1-6.3.3).
Since Hue shell is essentially Python shell, this should also be applicable to any versions, that I don't have a chance to test currently.

#### Running Hue Shell

Based on [Cloudera docs](https://docs.cloudera.com/documentation/enterprise/latest/topics/hue_use_shell_commands.html):

1. Set `HUE_CONF` dir:
```
export HUE_CONF_DIR="/var/run/cloudera-scm-agent/process/`ls -alrt /var/run/cloudera-scm-agent/process | grep HUE_SERVER | tail -1 | awk '{print $9}'`"
echo $HUE_CONF_DIR
```
2. Set necessary environment variables:
RHEL/CentOS:
```
for line in `strings /proc/$(lsof -i :8888|grep -m1 python|awk '{ print $2 }')/environ|egrep -v "^HOME=|^TERM=|^PWD="`;do export $line;done
```
Debian/Ubuntu:
```
for line in `strings /proc/$(lsof -i :8888|grep -m1 hue|awk '{ print $2 }')/environ|egrep -v "^HOME=|^TERM=|^PWD="`;do export $line;done
```
3. Run `hue shell` - if that doesn't work, execute from absolute path, like:
```
/opt/cloudera/parcels/CDH/lib/hue/build/env/bin/hue shell
```


#### Creating users

In a very basic way, if just one user is needed, can be done like:
```
from django.contrib.auth.models import User, Permission, Group
newuser = User.objects.create(username='alice', password='P@ssw0rd', email='alice@org.com', first_name='Alice', last_name='AliceAlice')
newuser.set_password('P@ssw0rd')
newuser.save()
findgroup = Group.objects.get(name='HDFS_ReadOnly')
findgroup.user_set.add(newuser.id)
```

But of course, that is usually not the case and we need to create a lot of users at once. Let's say we have a .csv file of future users, such as:
```
+----------+-----------+-----------------+-----------+----------------+
| Username | Password  |      Email      | FirstName |    LastName    |
+----------+-----------+-----------------+-----------+----------------+
| alice    | initPass1 | alice@org.com   | Alice     | AliceAlice     |
| bob      | initPass2 | bob@org.com     | Bob       | BobBob         |
| charlie  | initPass3 | charlie@org.com | Charlie   | CharlieCharlie |
| eve      | initPass4 | eve@org.com     | Eve       | EveEve         |
| ...      | ......... | ...........     | ...       | ......         |
| ...      | ......... | ...........     | ...       | ......         |
| ...      | ......... | ...........     | ...       | ......         |
| zoe      | initPassn | zoe@org.com     | Zoe       | ZoeZoe         |
+----------+-----------+-----------------+-----------+----------------+
```

In that case, first we can create a function to create user, then iterate through .csv file:
```
import csv
from django.contrib.auth.models import User, Permission, Group

### Function to create a user
def HueUserAdd(username, password, email, first_name, last_name):
    newuser = User.objects.create(username=username, password=password, email=email, first_name=first_name, last_name=last_name)
    newuser.set_password(password)
    newuser.save()
    findgroup = Group.objects.get(name='HDFS_ReadOnly') ### Obviously, we can grab this info from a .csv too
    findgroup.user_set.add(newuser.id)


### Function to run batch creation
def CreateUsersBatch(mycsvfile):
    with open(mycsvfile) as hueusers:
        getusers=csv.reader(hueusers, delimiter=';')
        for i in getusers:
            HueUserAdd(i[0], i[1], i[2], i[3], i[4])

```
And considering our .csv file is named "userlist.csv", we can now simply run `CreateUsersBatch("/path/to/userlist.csv")` from the same shell, and we are done.



#### Deactivating users

I am not looking into batch deletion of users here, since that can be easily done from Hue UI itself. However batch deactivation can come in handy:

```
rom django.contrib.auth.models import User, Permission, Group
def DeactivateUsers(username):
    theuser=User.objects.get(username=username)
    theuser.is_active=False
    theuser.save()
```

Considering we have a list of usernames to deactivates, you can just iterate through that list to deactivate them at once.
