## LDAP login 
## Если не работает синхронизация с LDAP - выключи и зайди под локальной записью.

``` OFF
sudo -u www-data php /srv/nextcloud/occ app:disable user_ldap
```

``` ON
sudo -u www-data php /srv/nextcloud/occ app:disable user_ldap
```
