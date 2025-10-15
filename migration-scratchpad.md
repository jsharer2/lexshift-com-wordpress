bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp option get siteurl && sudo wp option get home && sudo wp core version
https://lexshift.com
https://lexshift.com
6.8.3
bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp theme list
+--------------------+----------+--------+---------+----------------+-------------+
| name | status | update | version | update_version | auto_update |
+--------------------+----------+--------+---------+----------------+-------------+
| bluehost-blueprint | inactive | none | 1.0.0 | | on |
| inspiro | inactive | none | 2.1.4 | | on |
| onepress | active | none | 2.3.15 | | on |
| twentytwentyfive | inactive | none | 1.3 | | on |
| twentytwentyfour | inactive | none | 1.3 | | on |
| twentytwentythree | inactive | none | 1.6 | | on |
+--------------------+----------+--------+---------+----------------+-------------+
bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp plugin list
+-----------------------------------+----------+-----------+-----------+----------------+-------------+
| name | status | update | version | update_version | auto_update |
+-----------------------------------+----------+-----------+-----------+----------------+-------------+
| akismet | inactive | none | 5.5 | | on |
| all-in-one-wp-migration | inactive | none | 7.100 | | on |
| better-search-replace | active | none | 1.4.10 | | on |
| creative-mail-by-constant-contact | active | none | 1.6.9 | | on |
| duplicate-page | active | none | 4.5.5 | | on |
| elementor | active | none | 3.32.4 | | off |
| elementor-pro | active | none | 3.32.2 | | off |
| filebird | active | none | 6.4.9 | | off |
| olympus-google-fonts | active | none | 3.9.8 | | on |
| google-analytics-for-wordpress | active | none | 9.8.0 | | on |
| happy-elementor-addons | active | none | 3.20.1 | | off |
| ht-slider-for-elementor | active | none | 1.7.2 | | off |
| jetpack | active | none | 15.1.1 | | on |
| malcare-security | active | none | 6.02 | | off |
| media-sync | active | none | 1.4.8 | | off |
| onepress-plus | active | available | 2.3.8 | 2.3.10 | on |
| optinmonster | active | none | 2.16.21 | | on |
| royal-elementor-addons | active | none | 1.7.1035 | | off |
| wp-smushit | inactive | none | 3.22.1 | | off |
| svg-support | active | none | 2.5.14 | | off |
| header-footer-elementor | active | available | 2.5.2 | 2.6.1 | off |
| updraftplus | active | none | 2.25.8.26 | | on |
| w3-total-cache | active | none | 2.8.13 | | off |
| wordfence | active | none | 8.1.0 | | off |
| wp-crontrol | active | none | 1.19.2 | | off |
| wpforms-lite | active | none | 1.9.8.2 | | on |
| wp-mail-smtp | active | none | 4.6.0 | | off |
| wp-staging-pro | active | none | 6.3.2 | | off |
| wp-staging | active | none | 4.3.2 | | off |
| wordpress-seo | inactive | none | 26.1.1 | | on |
| sso | must-use | | 0.5 | | off |
| wp-staging-optimizer | must-use | | 1.6.0 | | off |
| advanced-cache.php | dropin | | | | off |
+-----------------------------------+----------+-----------+-----------+----------------+-------------+
bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp menu list
+---------+---------+---------+-----------+-------+
| term_id | name | slug | locations | count |
+---------+---------+---------+-----------+-------+
| 25 | Primary | primary | primary | 7 |
+---------+---------+---------+-----------+-------+
bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp menu item list primary
+-------+--------+----------+-----------------+----------+
| db_id | type | title | link | position |
+-------+--------+----------+-----------------+----------+
| 1239 | custom | Products | /home/#products | 1 |
| 28 | custom | Services | /home/#services | 2 |
| 27 | custom | About | /home/#about | 3 |
| 1205 | custom | Plans | /home/#plans | 4 |
| 1726 | custom | Team | /home/#team | 5 |
| 31 | custom | News | /home/#news | 6 |
| 32 | custom | Contact | /home/#contact | 7 |
+-------+--------+----------+-----------------+----------+
bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$ sudo wp post list --post_type=page --fields=post_title,guid
+------------------------------------------+----------------------------------------------------------------------------------------+
| post_title | guid |
+------------------------------------------+----------------------------------------------------------------------------------------+
| Andrew Borodenko - DOO | https://www.new.lexshift.com/?page_id=2044 |
| Shirley Qin - CPO | https://www.new.lexshift.com/?page_id=2041 |
| Terry Wing - CTO | https://www.new.lexshift.com/?page_id=2035 |
| Jason C. Stearns - CCSO | https://www.new.lexshift.com/?page_id=2030 |
| John J. Isaza - COO | https://18.222.78.21/?page_id=1910 |
| Implementation Support | https://18.222.78.21/?page_id=1838 |
| Artificial Intelligence Compliance | https://18.222.78.21/?page_id=1832 |
| General Counsel | https://18.222.78.21/?page_id=1830 |
| Privacy Compliance | https://18.222.78.21/?page_id=1826 |
| Records And Information Management (RIM) | https://18.222.78.21/?page_id=1818 |
| Illuminate | https://18.222.78.21/?page_id=1788 |
| DEMO | https://18.222.78.21/?page_id=1751 |
| Jeffrey C. Sharer - CEO | https://18.222.78.21/?page_id=1711 |
| Sharer Law | https://18.222.78.21/?page_id=1660 |
| Isaza Law, PC | https://18.222.78.21/?page_id=1649 |
| Privacy Policy | http://box5434/cgi/addon_GT.cgi?s=GT::WP::Install::Cpanel+%28fblsgamy%29+-+127.0.0.1+% |
| | 5Bnocaller%5D/?page_id=3 |
| WPForms Preview | https://demos.famethemes.com/onepress-plus/wpforms-preview/ |
| Our Projects | https://demos.famethemes.com/onepress-plus/?page_id=1327 |
| Shop | http://demos.famethemes.com/onepress-plus/shop/ |
| Cart | http://demos.famethemes.com/onepress-plus/cart/ |
| Checkout | http://demos.famethemes.com/onepress-plus/checkout/ |
| My Account | http://demos.famethemes.com/onepress-plus/my-account/ |
| Service Title #6 | http://demos.famethemes.com/onepress-plus/?page_id=1207 |
| Service Title #5 | http://demos.famethemes.com/onepress-plus/?page_id=1206 |
| Service Title #4 | http://demos.famethemes.com/onepress-plus/?page_id=26 |
| Orchestrate Your Policy | http://demos.famethemes.com/onepress-plus/?page_id=25 |
| Service Title #2 | http://demos.famethemes.com/onepress-plus/?page_id=24 |
| Orchestrate | http://demos.famethemes.com/onepress-plus/?page_id=23 |
| Our Vision | http://demos.famethemes.com/onepress-plus/?page_id=22 |
| Our Mission | http://demos.famethemes.com/onepress-plus/?page_id=21 |
| Our History | http://demos.famethemes.com/onepress-plus/?page_id=17 |
| News | http://demos.famethemes.com/onepress-plus/?page_id=5 |
| Home | http://demos.famethemes.com/onepress-plus/?page_id=4 |
+------------------------------------------+----------------------------------------------------------------------------------------+
bitnami@ip-172-26-13-63:/opt/bitnami/wordpress$
