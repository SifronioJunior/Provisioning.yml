---
- hosts: wordpress
  handlers:
    - name: restart apache
      service:
        name: apache2
        state: restarted
      become: yes
  tasks:
    - name: Install dependencias
      ansible.builtin.apt:
        pkg:
        - apache2
        - ghostscript
        - libapache2-mod-php 
        - php 
        - php-bcmath 
        - php-curl 
        - php-imagick 
        - php-intl 
        - php-json 
        - php-mbstring 
        - php-mysql 
        - php-xml 
        - php-zip
        state: latest
        update_cache: yes
      become: yes
    - name: Create a directory if it does not exist
      ansible.builtin.file:
        path: /srv/www
        state: directory
        owner: www-data
        group: www-data
      become: yes
    - name: Unarchive a file that needs to be downloaded (added in 2.0)
      ansible.builtin.unarchive:
        src: https://wordpress.org/latest.tar.gz
        dest: /srv/www
        remote_src: yes
      become: yes
    - name: Copy file with owner and permissions
      ansible.builtin.copy:
        src: files/wordpress.conf
        dest: /etc/apache2/sites-available/000-default.conf
      become: yes
      notify:
        - restart apache
    - name: Copy file with owner and permissions
      ansible.builtin.copy:
        src: '/srv/www/wordpress/wp-config-sample.php'
        dest: '/srv/www/wordpress/wp-config.php'
        force: no
        remote_src: yes
      become: yes
    - name: configure wp-config with database
      ansible.builtin.replace:
        path: '/srv/www/wordpress/wp-config.php'
        regexp: "{{ item.regexp }}"
        replace: "{{ item.replace }}"
      with_items:
      - { regexp: "database_name_here", replace: "wordpress_db"}
      - { regexp: "username_here", replace: "wordpress_user"}
      - { regexp: "password_here", replace: "12345"}
      - { regexp: "localhost", replace: "192.168.1.99"}
      become: yes
    - name: Replace a localhost entry searching for a literal string to avoid escaping
      ansible.builtin.lineinfile:
        path: /srv/www/wordpress/wp-config.php
        search_string: "{{ item.search_string }}"
        line: "{{ item.line }}"
      with_items:
      - { search_string: "define( 'AUTH_KEY',         'put your unique phrase here' );", line: "define('AUTH_KEY',         '074-`_ZG2%V@0*bGmEDYTjiTGi|c0O+;&;#=cuxX|rz3(Nnu~@qsDN>35/IoB 7N');"}
      - { search_string: "define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );", line: "define('SECURE_AUTH_KEY',  'zVdRZa-0Lh)<!JpZBlh#xW{8c.%eDU@l4mg^a_|/2Si+[6{y$eA/o%tZ^)Mz|]gk');"}
      - { search_string: "define( 'LOGGED_IN_KEY',    'put your unique phrase here' );", line: "define('LOGGED_IN_KEY',    'D8,[S-6?GD&.9?|2t9#qL|G<S63jbdSI_IoqPw-]d+q} y[)<;Wqt5F[lxv8-I<4');"}
      - { search_string: "define( 'NONCE_KEY',        'put your unique phrase here' );", line: "define('NONCE_KEY',        '_ZPwMt tr#^sZyQ><!7nU!0]B$yWU6j47|cVb1JSqRuI!Q7;d:kDbQoyO9!dH`f&');"}
      - { search_string: "define( 'AUTH_SALT',        'put your unique phrase here' );", line: "define('AUTH_SALT',        'jCT&<l]DS/<t^1_(p?AO$WFC&/|+5Dd04sT9+VZt.,LO9?GYL#+,Hm3F@=SK]~D1');"}
      - { search_string: "define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );", line: "define('SECURE_AUTH_SALT', '`(u_GH2U^A_ErO0X9VD?V->k+{>8krM-)& [gVEzD{_L!}g+zkLEn4+*Ht<)Pq%?');"}
      - { search_string: "define( 'LOGGED_IN_SALT',   'put your unique phrase here' );", line: "define('LOGGED_IN_SALT',   'sK=-/D)y<T8Z3DNnDXLk+A+IUQ]Ba-,5i3S/`h-%a`%(C1fxT#Wi}*dtUL5qm:Zv');"}
      - { search_string: "define( 'NONCE_SALT',       'put your unique phrase here' );", line: "define('NONCE_SALT',       '.I[c0 ;2)}3>}O*sCJfda{D_TJmPFuGWd4{9J-OJEG;RValW.A,?Za/V:|1X%{r)');"}
      become: yes

- hosts: mysql
  handlers:
    - name: restart mysql
      service:
        name: mysql
        state: restarted
      become: yes
  tasks:
    - name: Install dependencias
      ansible.builtin.apt:
        pkg:
        - mysql-server
        - python3-pymysql
        state: latest
        update_cache: yes
      become: yes
    - name: Create a new database with name 'wordpress_db'
      community.mysql.mysql_db:
        name: wordpress_db
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
      become: yes
    - name: Create database user with name 'wordpress_user' and password '12345' with all database privileges
      community.mysql.mysql_user:
        name: 'wordpress_user'
        password: '12345'
        priv: 'wordpress_db.*:SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER'
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
        host: '{{ item }}'
      with_items:
      - 'localhost'
      - '127.0.0.1'
      - '192.168.1.99'
      become: yes
    - name: configure database
      ansible.builtin.replace:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: "127.0.0.1"
        replace: "0.0.0.0"
      become: yes
      notify:
        - restart mysql
        
