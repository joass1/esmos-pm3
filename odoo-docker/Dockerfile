FROM odoo:18.0
USER root
RUN pip3 install --no-cache-dir --break-system-packages num2words xlwt
RUN mkdir -p /mnt/extra-addons/oca-helpdesk
COPY custom-addons/oca-helpdesk/ /mnt/extra-addons/oca-helpdesk/
RUN echo "[options]" > /etc/odoo/odoo.conf && \
    echo "addons_path = /mnt/extra-addons,/mnt/extra-addons/oca-helpdesk,/usr/lib/python3/dist-packages/odoo/addons" >> /etc/odoo/odoo.conf && \
    echo "data_dir = /var/lib/odoo" >> /etc/odoo/odoo.conf && \
    echo "limit_time_cpu = 600" >> /etc/odoo/odoo.conf && \
    echo "limit_time_real = 1200" >> /etc/odoo/odoo.conf && \
    echo "db_maxconn = 64" >> /etc/odoo/odoo.conf && \
    echo "workers = 2" >> /etc/odoo/odoo.conf && \
    echo "max_cron_threads = 1" >> /etc/odoo/odoo.conf && \
    echo "admin_passwd = 214Odoo" >> /etc/odoo/odoo.conf
RUN chown -R odoo:odoo /mnt/extra-addons /etc/odoo
USER odoo
