# Odoo en profondeur (Suite 3)


# Les tableaux de bord (Reporting) dans Odoo:
- Un tableau de bord n'est lié qu'a un seul modéle.
- Pour contourner cette restriction le modéle du reporting est construit à partir d'une vue base de donées.


```Python
from odoo import api, fields, models, tools


class PurchaseReport(models.Model):
    _name = "purchase.report"
    _description = "Purchases Orders"
    _auto = False
    _order = 'date_order desc, price_total desc'

    date_order = fields.Datetime('Order Date', readonly=True, help="Date on which this document has been created", oldname='date')
    state = fields.Selection([
        ('draft', 'Draft RFQ'),
        ('sent', 'RFQ Sent'),
        ('to approve', 'To Approve'),
        ('purchase', 'Purchase Order'),
        ('done', 'Done'),
        ('cancel', 'Cancelled')
        ], 'Order Status', readonly=True)
    ...
    
    @api.model_cr
    def init(self):
        tools.drop_view_if_exists(self._cr, 'purchase_report')
        self._cr.execute("""
            create view purchase_report as (
                WITH currency_rate as (%s)
                select
                ...
            )
        """ % self.env['res.currency']._select_companies_rates())
```

- Actuellement, Odoo prend en charge quatre modes d'affichages:
    - **Pivot**
    - **Bar**
    - **Pie**
    - **Line**

#### Mesure et dimension
- **Mesure:** une mesure est un champ qui peut être agrégé. Chaque champ de type entier ou float (sauf le champ id) peut être utilisé comme mesure.
- Deux modes d'agrégation sont disponibles: `sum` et `avg`.

```Python
...
    price_average = fields.Float('Average Price', readonly=True, group_operator="avg")
...
```
- **dimension:** Une dimension est un champ qui peut être groupé. Cela signifie à peu près tous les champs non numériques.
- Pour mieux comprendre comment Odoo sélectionne si un champ est une dimension ou une mesure dans la vue Graph et Pivot:
```Javascript
    prepare_fields: function (fields) {
        var self = this;
        this.fields = fields;
        _.each(fields, function (field, name) {
            if ((name !== 'id') && (field.store === true)) {
                if (field.type === 'integer' || field.type === 'float' || field.type === 'monetary') {
                    self.measures[name] = field;
                }
            }
        });
        this.measures.__count__ = {string: _t("Count"), type: "integer"};
    },
```

```Javascript
    prepare_fields: function (fields) {
        var self = this,
            groupable_types = ['many2one', 'char', 'boolean',
                               'selection', 'date', 'datetime'];
        this.fields = fields;
        _.each(fields, function (field, name) {
            if ((name !== 'id') && (field.store === true)) {
                if (field.type === 'integer' || field.type === 'float' || field.type === 'monetary') {
                    self.measures[name] = field;
                }
                if (_.contains(groupable_types, field.type)) {
                    self.groupable_fields[name] = field;
                }
            }
        });
        this.measures.__count__ = {string: _t("Count"), type: "integer"};
    },
```

- Pour définir la vue Graph et Pivot:

```XML
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record model="ir.ui.view" id="view_purchase_order_pivot">
        <field name="name">product.month.pivot</field>
        <field name="model">purchase.report</field>
        <field name="arch" type="xml">
            <pivot string="Purchase Orders Statistics" disable_linking="True">
                <field name="partner_id" type="row"/>
                <field name="date_order" interval="month" type="col"/>
                <field name="price_total" type="measure"/>
                <field name="unit_quantity" type="measure"/>
                <field name="price_average" type="measure"/>
            </pivot>
        </field>
    </record>
    <record model="ir.ui.view" id="view_purchase_order_graph">
        <field name="name">product.month.graph</field>
        <field name="model">purchase.report</field>
        <field name="arch" type="xml">
            <graph string="Purchase Orders Statistics">
                <field name="partner_id" type="row"/>
                <field name="date_order" interval="month" type="col"/>
                <field name="price_average" type="measure"/>
            </graph>
        </field>
    </record>
    ...
</odoo>
```

- Malheureusement, le Repporting dans Odoo possède quelques points négatifs:
- Le fait de ne pas utiliser l'ORM pour construire le modèle de rapport, donc les utilisateurs peuvent accéder aux données qu'ils ne devraient pas voir.
- Dans certains cas, les données agrégées n'ont aucun sens, Exemple: le prix moyen unitaire.
- Il faut être très prudent avec les devises et les taux de change.
- Un outils très intéréssant pour construire des tableau de bord avancés: **MIS Builder**
    - https://github.com/OCA/mis-builder/tree/10.0
    - https://www.youtube.com/watch?v=0PpxGAf2l-0


## Wizard dans Odoo
- Les assistants sont généralement utilisés pour les butes suivants:
    * Pour recueillir des informations et permettre aux utilisateurs de faire des choix.
    * Pour effectuer une action qui ne peut pas être annulée en cliquant sur Retour ou Annuler.
- Les assistants dans odoo utilisent un **TransiantModel**.
- Les règles de sécurité ne s'applique pas aux assistants.
- généralement un assistant est toujours lié à un ou plusieurs modèles principaux.





# Déploiement Odoo

- La version qui devrait être utilisée pour le déploiement d'Odoo sera la version 16.04 LTS ou plus récente.
- La plupart des étapes à venir ont déjà été expliquées lorsque nous avons configuré notre environnement de développement.
- La plus grande différence: dans un environnement de production, nous nous concentrons plus sur la sécurité des fichiers (en particulier les fichiers de configuration).


1- Mettre à jour les packages Ubuntu:
```Shell
sudo apt update && sudo apt upgrade
```

2- Créer un utilisateur système `Odoo` qui possédera et exécutera l'application:
```Shell
sudo adduser --system --home=/opt/odoo --group odoo
```

3- Installer les packages
```Shell
sudo apt install git python-pip postgresql postgresql-server-dev-9.5 python-all-dev python-dev python-setuptools libxml2-dev libxslt1-dev libevent-dev libsasl2-dev libldap2-dev pkg-config libtiff5-dev libjpeg8-dev libjpeg-dev zlib1g-dev libfreetype6-dev liblcms2-dev liblcms2-utils libwebp-dev tcl8.6-dev tk8.6-dev python-tk libyaml-dev fontconfig
```

4- Clonner Odoo dans le nouveau répertoire `/opt/odoo`

```Shell
sudo git clone https://www.github.com/odoo/odoo --depth 1 --branch 10.0 --single-branch /opt/odoo
```

5- Créer un environnement virtuel:

```Shell
cd /opt/odoo/
mkvirtualenv odoo_genisoft -a .
```

6- Installer les dépendances:

```Shell
pip install -r requirements.txt
```


7- Installer `Nodejs` et `Lessc`:

```Shell
cd
sudo curl -sL https://deb.nodesource.com/setup_4.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g less less-plugin-clean-css
```

8- Installer La bonne version de la librairie `Wkhtmltopdf`:

```Shell
cd /tmp
sudo wget https://downloads.wkhtmltopdf.org/0.12/0.12.1/wkhtmltox-0.12.1_linux-trusty-amd64.deb
sudo dpkg -i wkhtmltox-0.12.1_linux-trusty-amd64.deb
sudo cp /usr/local/bin/wkhtmltopdf /usr/bin
sudo cp /usr/local/bin/wkhtmltoimage /usr/bin
```

10- Configuration du serveur Odoo:

```Shell
cd
sudo cp /opt/odoo/debian/odoo.conf /etc/odoo-server.conf
```

```text
[options]
admin_passwd = admin
db_host = False
db_port = False
db_user = odoo
db_password = FALSE
addons_path = /opt/odoo/addons
;Uncomment the following line to enable a custom log
;logfile = /var/log/odoo/odoo-server.log
xmlrpc_port = 8069
```

11- Ajouter un service Odoo:
```Shell
sudo nano /lib/systemd/system/odoo-server.service
```

```text
[Unit]
Description=Odoo Open Source ERP and CRM
Requires=postgresql.service
After=network.target postgresql.service

[Service]
Type=simple
PermissionsStartOnly=true
SyslogIdentifier=odoo-server
User=odoo
Group=odoo
ExecStart=/home/genisoft/.virtualenvs/odoo_genisoft/bin/python /opt/odoo/odoo-bin --config=/etc/odoo-server.conf --addons-path=/opt/odoo/addons/
WorkingDirectory=/opt/odoo/
StandardOutput=journal+console

[Install]
WantedBy=multi-user.target
```

12- Sécuriser les fichiers:
```CMD
sudo chmod 755 /lib/systemd/system/odoo-server.service
sudo chown root: /lib/systemd/system/odoo-server.service

sudo chown -R odoo: /opt/odoo/

sudo chown odoo:root /var/log/odoo

sudo chown odoo: /etc/odoo-server.conf
sudo chmod 640 /etc/odoo-server.conf
```

13- Activer le service Odoo:
```CMD
sudo systemctl enable odoo-server
```


# Fin

- Apprendre le Git et les platforms de collaboration tel que: Gitlab, github, bitbucket.
- Apprendre les nouvelles technologies au tour du Devops: CI et CD (Continuous Integration et Continuous Deployment).
- Apprendre Python en d'hors d'Odoo: Flask, Django, PIPY, ...
- Apprendre Postgresql.
- Créer des comptes sur twitter, github, être un membre OCA (si possible).
