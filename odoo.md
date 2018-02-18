## Odoo en profondeur


![Interface Odoo](/images/odoo_architecture.png)



### Record and Recordsets
- Le but est de vraiment comprendre la signification de: **@api.model**, **@api.multi**, ...
- Chaque modèle dans Odoo est un **singleton**: Une et une seule instance dans l'environnement
- Dans python, chaque objet a une référence, qui peut avoir une représentation lisible par l'homme
```python
...
class BaseModel(object):
    ...
    def __add__(self, other):
        """ Return the concatenation of two recordsets. """
        if not isinstance(other, BaseModel) or self._name != other._name:
            raise TypeError("Mixing apples and oranges: %s + %s" % (self, other))
        return self.browse(self._ids + other._ids)
    ...
    def __str__(self):
        return "%s%s" % (self._name, getattr(self, '_ids', ""))

    __repr__ = __str__
    ...

    def __getitem__(self, key):
        """ If ``key`` is an integer or a slice, return the corresponding record
            selection as an instance (attached to ``self.env``).
            Otherwise read the field ``key`` of the first record in ``self``.

            Examples::

                inst = model.search(dom)    # inst is a recordset
                r4 = inst[3]                # fourth record in inst
                rs = inst[10:20]            # subset of inst
                nm = rs['name']             # name of first record in inst
        """
        if isinstance(key, basestring):
            # important: one must call the field's getter
            return self._fields[key].__get__(self, type(self))
        elif isinstance(key, slice):
            return self._browse(self.env, self._ids[key])
        else:
            return self._browse(self.env, (self._ids[key],))

    def __setitem__(self, key, value):
        """ Assign the field ``key`` to ``value`` in record ``self``. """
        # important: one must call the field's setter
        return self._fields[key].__set__(self, value)
    ...
```

- Par conséquent: Le **self** c'est l'instance courante du modèle qui est unique dans l'environnement, elle contient un ou plusieurs enregistrements:
```Python
class AModel(models.Model):

    _name = 'a.model'

    def a_method(self):
        # self can be anywhere between 0 records and all records in the
        # database
        self.do_operation()

    def do_operation(self):
        print self # => a.model(1, 2, 3, 4, 5)
        for record in self:
            print record # => a.model(1), then a.model(2), then a.model(3), ...
            print record.name # => u'Nom 1'
```


- Ceci nous permettra de mettre à jour les valeurs des champs d'enregistrements comme suit:
```Python
    #3 * len(records) database updates
    for record in records:
        record.a = 1
        record.b = 2
        record.c = 3

    # len(records) database updates
    for record in records:
        record.write({'a': 1, 'b': 2, 'c': 3})

    # 1 database update
    records.write({'a': 1, 'b': 2, 'c': 3})
```

- Les enregistrements sont représentés dans un ensemble. Toutes les actions disponibles sur les ensembles peuvent être effectuées sur des enregistrements:
```Python
    ...
    record_ids = record_subset_ids_1 + record_subset_ids_2 # => model(1, 2, 3) + model(2, 4, 5) = model(1, 2, 3, 4, 5)
    records_ids |= record_subset_ids_3 # =>  model(1, 2, 3, 4, 5) |= model(6) ==> model(1, 2, 3, 4, 5, 6)
    record_subset_ids_1 & record_subset_ids_2 # => model(1, 2, 3) & model(2, 4, 5) = model(2)
    ...
```

- Odoo fournit également des méthodes d'aide pour transformer des enregistrements:
    - Ordonner un ensemble d'enregistrements dans un certain ordre.
    - Prendre un sous-ensemble des enregistrements satisfaisant certaines conditions.
    - Déduire certaines valeurs.
```Python
    # returns a list of names
    records.mapped('name')

    # returns a recordset of partners
    record.mapped('partner_id')

    # returns the union of all partner banks, with duplicates removed
    record.mapped('partner_id.bank_ids')

    # sort records by name
    records.sorted(key=lambda r: r.name)

    # only keep records whose company is the current user's
    records.filtered(lambda r: r.company_id == user.company_id)

    # only keep records whose partner is a company
    records.filtered("partner_id.is_company")
```


- Conclusion:
```Python
from odoo import models, fields, api
class AModel(models.Model):
    _name = 'a.model.name'

    @api.model
    _get_default_method(self):
        print self # ===> a.model.name()
    
    @api.multi
    def _action_do_something(self):
        print self # ===> a.model.name(1, 2, 3, 4, ...)

    @api.multi
    def _action_do_something(self):
        self.ensure_one()
        print self # ===> a.model.name(35)
```

- **Environnement**: stocke diverses données contextuelles utilisées par l'ORM:
    - Le curseur de la base de données (pour les requêtes de base de données) ==> **self.env.cr**.
    - L'utilisateur actuel (pour la vérification des droits d'accès) ==> **self.env.user**.
    - Le contexte actuel (stockage de métadonnées arbitraires) ==> **self.env.context**
    - L'environnement stocke également les caches.
```Shell
>>> records.env
<Environment object ...>
>>> records.env.user
res.user(3)
>>> records.env.cr
<Cursor object ...)
>>> self.env['res.partner']
res.partner
```

- L'environnement peut être personnalisé à partir d'un recordset. Cela renvoie une nouvelle version utilisant l'environnement modifié.
```Python
    # create partner object as administrator
    env['res.partner'].sudo().create({'name': "A Partner"})

    # There is an OCA module that runs the function with sudo keeping the same user
    env['res.partner'].suspend_security().create({'name': "A Partner"})

    # look for partner, or create one with specified timezone if none is
    # found
    env['res.partner'].with_context(tz=a_tz).find_or_create(email_address)
```



## Communication entre back-end et Postgresql.
- Nous avons déjà parlé des fonctions CRUD: create (), read (), write (), unlink ()
- Mais les bases de données offrent beaucoup plus: contraintes, triggers, savepoints, rollback, ...
- Comment Odoo traite tout ça?

- **Contraintes d'intégrités**: Il est très important de garder la base de données cohérente et de veiller à l'intégrité des données.
- Dans Odoo, nous avons les contraintes suivantes:
    - Au niveau des attributs: Required, ID, Many2one, _sql_constraints.
    - Au niveau du model: Vérification des données avant l'insertion.

- **Required**: Sera converti en une contrainte `NOT NULL`.
```python
    ...

    name = fields.Char(
        string='Nom',
        required=True,
        ...
    )
    ...
    # Dans la DB
    ...
    name varchar not null,
    ...
```


- Habituellement, avec un champ obligatoire, nous utilisons l'opérateur par défaut, pour obtenir une valeur par défaut pour ce champ.
- Les valeurs par défaut peuvent être exprimées de deux façons: **valeur statique** ou une **fonction** renvoyant une valeur
```Python
    @api.model
    def _get_default_name(self):
        return self.env['ir.sequence'].get('purchase.request')

    name = fields.Char(
        string=...,
        default=_get_default_name,
    )
    state = fields.Selection(
        selection=[('draft', 'Draft'), ...],
        default='draft',
    )
    requested_by = fields.Many2one(
        comodel_name='res.users',
        default=lambda self: self.env.user,
    )
```

- **ID**: Chaque modèle (table) a une clé primaire unique.
```SQL
id serial not null
    constraint account_account_pkey
	primary key,
```

- **Many2one**: Sera exprimé en clé étrangère.
```Python
create_uid = fields.Many2one('res.users', string='Created by', ...)
```
```SQL
create_uid integer
    constraint account_account_create_uid_fkey
        references res_users on delete set null,
```
- Notez que l'option **`ondelete`** doit être explicitement définie. Par défaut elle est égal à
**`not null`**.


- Vérification des données avant l'insertion: **`api.constraint(field_list)`**
- **Remarque**: les champs doivent être sauvegardés dans la base de données.
```Python
    ...
    @api.multi
    @api.constrains('name', 'description')
    def _check_description(self):
        for rec in self:
            if rec.name == rec.description:
                raise ValidationError(
                    _("Fields name and description must be different."))
    ...
```

- **Triggers**: dans Odoo, ils sont exprimés en utilisant des champs calculés. Ils peuvent être sauvegardés en db ou calculés à la volée chaque fois qu'ils sont utilisés.
```Python
    discount_value = fields.Float(
        compute='_apply_discount',
        store=True,
        readonly=True,
    )
    total = fields.Float(
        compute='_apply_discount',
        store=True,
        readonly=True,
    )

    @api.multi
    @depends('value', 'discount')
    def _apply_discount(self):
        for record in self:
            # compute actual discount from discount percentage
            discount = record.value * record.discount
            record.discount_value = discount
            record.total = record.value - discount
```

- Un cas particulier de champs calculés sont des champs liés (proxy), qui fournissent la valeur d'un sous-champ sur l'enregistrement en cours.
```Python
nickname = fields.Char(related='user_id.partner_id.name', store=True)
```


- Il y a quelques modèles spécifiques dans Odoo qui se comportent d'une manière différente que d'habitude (le modèle est converti en table dans la base de donées.)
- **TransientModel**: temporairement persisté et régulièrement nettoyé. Généralement utilisé pour les wizards.
```Python
    class TransientModel(BaseModel):
        # Model super-class for transient records, meant to be temporarily
        # persisted, and regularly vacuum-cleaned.
        # A TransientModel has a simplified access rights management,
        # all users can create new records, and may only access the
        # records they created. The super-user has unrestricted access
        # to all TransientModel records.

        _auto = True
        _register = False # not visible in ORM registry, meant to be python-inherited only
        _transient = True
```

- **AbstractModel**: Pour être hérité par des modèles réguliers (Models ou TransientModels) mais non destiné à être utilisable seul ou persistant.
```Python
    class AbstractModel(BaseModel):
        # Abstract Model super-class for creating an abstract class meant to be
        # inherited by regular models (Models or TransientModels) but not meant to
        # be usable on its own, or persisted.
        # Technically: we do not  want to make AbstractModel the super-class of
        # Model or BaseModel because it would not make sense to put the main
        # definition of persistence methods such as create() in it, and still we
        # should be able to override them within an AbstractModel.

        _auto = False # don't create any database backend for AbstractModels
        _register = False # not visible in ORM registry, meant to be python-inherited only
        _transient = False
```


- **Vue materialisée**: c'est des modèles réguliers dont la table est crée manuellement:
```Python
class PurchaseReport(models.Model):

    _name = 'silog.purchase.report'
    _auto = False
    _order = 'date_order desc, price_total desc'

    partner_id = fields.Many2one(
        string='Fournisseur',
        comodel_name='res.partner',
        readonly=True,
    )
    date_order = fields.Datetime(
        string=u"Date de création",
        readonly=True
    )
    ...

    def init(self, cr):
        tools.sql.drop_view_if_exists(cr, 'silog_purchase_report')
        # pylint:disable=sql-injection
        cr.execute("""
            create or replace view silog_purchase_report as (
                WITH currency_rate as (%s)
                select
                    min(l.id) as id,
                    s.date_order,
                    s.date_approve,
                    s.partner_id as partner_id,
                    l.public_contract_id,
                    ...
            )"""...)
```


### Les domaines
- Les domaines sont des valeurs qui permettent d'exprimer des conditions sur des enregistrements.
- Un domaine est une liste de critères, chaque critère étant un triplet (ou une liste ou un tuple) of **(field_name, operator, value)**.
```Python
[('product_type', '=', 'service'), ('unit_price', '>', 1000)]
```
- Un domaine est construit de la manière suivante:
```Python
    ( A or B ) AND ( C or D or E )
    # First simplification:
    AND ( A or B ) ( C or D or E )
    #
    AND ( or A B ) ( C or D or E )
    #
    AND ( or A B ) ( or C ( D or E ) )
    #
    AND or A B or C or D E
    # Final Syntax
    [ '&', '|', (A), (B), '|', (C), '|', (D), (E) ]
```


## MAGIC NUMBERS

- **One2many** et **Many2many** utilisent un format "commands" spécial pour manipuler l'ensemble des enregistrements stockés dans / associés au champ.
    - (0, _, values): Ajouter un nouveau record.
    
    - (1, id, values): mettre à jour les valeur du record avec le ID égale à `id`.

    - (2, id, _): Supprimer le record avec le ID égale à `id`. Ne peut pas être utilisé dans `create ()`.

    - (3, id, _): Supprimer le record avec le ID égale à `id` mais ne le supprime pas de la BD. Ne peut pas être utilisé avec `One2many`. Ne peut pas être utilisé dans `create ()`.

    - (4, id, _): Ajouter Un record déjà existant dans la base.

    - (5, _, _): Equivalente à 3. mais pour tous les enregistrements.

    - (6, _, ids): Combine entre le 5 et le 4.


```Python
    self.order.write({'order_line': [
            (0, 0, {
                'product_id': self.auxiliary_product_3.id,
                'product_tmpl_id': self.auxiliary_product_3.product_tmpl_id.id,
                'product_qty': 3,
                'date_planned': fields.Datetime.now(),
                'product_uom': self.auxiliary_product_3.uom_id.id,
                'name': self.auxiliary_product_3.supplier_product_name,
                'price_unit': 1000.0
            })
        ]})

    wizard = self.env['silog.add.auxiliary.to.purchase'].create({
            'record_id': order_line.id,
            'record_model': order_line._name,
            'auxiliary_product_ids':
            [(6, False, [auxiliary_product2.id, auxiliary_product.id])]
        })
```
```XML
    <record id="silog_purchase_reader_group" model="res.groups">
        <field name="name">Support - Lecteur Achat</field>
        <field name="category_id" ref="silog_base_security.silog_security_category"/>
        <field name="users" eval="[(4, ref('base.user_root'))]"/>
        <field name="implied_ids"
               eval="[(6, 0, [ref('silog_purchase.silog_purchase_lecteur_group')])]"/>
    </record>
```



## Communication entre front-end et back-end

- La communication est établie via des appels d'API  **XML-RPC / JSON-RPC**.

![Sequence](/images/sequence.png)


- **ListView**: Peut représenter une vue principale ou imbriquée dans une `FormView`. Utilisée pour représenter un ensemble d'enregistrements
```XML
    <record id="view_quotation_tree" model="ir.ui.view">
        ...
        <field name="arch" type="xml">
            <tree string="Quotation" decoration-bf="message_needaction==True" decoration-muted="state=='cancel'">
                <field name="message_needaction" invisible="1"/>
                <field name="name" string="Quotation Number"/>
                <field name="date_order"/>
                <field name="partner_id"/>
                <field name="user_id"/>
                <field name="amount_total" sum="Total Tax Included" widget="monetary"/>
                <field name="state"/>
            </tree>
        </field>
    </record>
```
- **FromView**: Peut représenter une vue principale ou imbriquée dans une `FormView`. Utilisée pour représenter un enregistrement
```XML
    <form string="Sales Order">
        <header>
            <button name="%(action_view_sale_advance_payment_inv)d" string="Create Invoice"
                    type="action" class="btn-primary"
                    attrs="{'invisible': [('invoice_status', '!=', 'to invoice')]}"/>
            <button name="action_quotation_send" string="Send by Email" type="object" states="draft" class="btn-primary"/>
            ...
            <field name="state" widget="statusbar" statusbar_visible="draft,sent,sale,done"/>
        </header>
        <sheet>
            ...
            <div class="oe_title">
                <h1>
                    <field name="name" readonly="1"/>
                </h1>
            </div>
            ...
            <group>
                <group>
                    <field name="partner_id" domain="[('customer','=',True)]" context="{'search_default_customer':1, 'show_address': 1}" options='{"always_reload": True}'/>
                    ...
                </group>
                <group>
                    <field name="date_order" invisible="1"/>
                    <field name="pricelist_id" groups="product.group_sale_pricelist"/>
                    ...
                </group>
            </group>
            <notebook>
                <page string="Order Lines">
                    ...
                </page>
                ...
            </notebook>
            ...
        </sheet>
    </form>
```
- https://www.odoo.com/documentation/11.0/reference/views.html#forms


![Form](/images/form_view.png)


- **`col`** vs **`colspan`**: Chaque champ prend deux positions (colonnes), une pour le libellé et l'autre pour l'input
```XML
<field name="inpt4" colspan="4"/>
<field name="input" />
<field name="inpt2" />
┌───────┬───────────────────────┐
│ labl4 │ inpt4_______________  │
├───────┼───────┬───────┬───────┤
│ label │ input │ labl2 │ inpt2 │
└───────┴───────┴───────┴───────┘
```
```XML
<group col="2" colspan="2">
    <field name="a" />
    <field name="b" />
</group>
<group col="6" colspan="2">
    <field name="d" />
    <field name="e" />
    <field name="f" />
</group>
│       │       │                │                │
├───────┴───────┼────────────────┴────────────────┤
│ ┌────┬───┐    │  ┌────┬───┬────┬───┬────┬───┐   │
│ │ lb │ a │    │  │ lb │ d │ lb │ e │ lb │ f │   │
│ ├────┼───┤    │  └────┴───┴────┴───┴────┴───┘   │
│ │ lb │ b │    │                                 │
│ └────┴───┘    │                                 │
├───────┬───────┼────────────────┬────────────────┤
│       │       │                │                │
```


- **Kanban Views**: Permet d'utiliser du HTML/CSS (bootstrap) dans une vue Odoo via des templates QWEB.
```XML
    <kanban class="o_kanban_mobile">
        <field name="name"/>
        <field name="partner_id"/>
        ...
        <templates>
            <t t-name="kanban-box">
                <div t-attf-class="oe_kanban_card oe_kanban_global_click">
                    <div class="row">
                        <div class="col-xs-6">
                            <strong><span><t t-esc="record.partner_id.value"/></span></strong>
                        </div>
                        <div class="col-xs-6 pull-right text-right">
                            <strong><field name="amount_total" widget="monetary"/></strong>
                        </div>
                    </div>
                    <div class="row">
                       ...
                    </div>
                </div>
            </t>
        </templates>
    </kanban>
```

- **Embeded Views**: Utilisées géneralement pour la représentation des champs `One2many`.
```XML
    ...
    <notebook>
        <page string="Order Lines">
            <field name="order_line" mode="tree,kanban"
                attrs="{'readonly': [('state', 'in', ('done','cancel'))]}">
                <form string="Sales Order Lines">
                    ...
                </form>
                <tree string="Sales Order Lines" editable="bottom" decoration-info="invoice_status=='to invoice'">
                    ...
                </tree>
                <kanban class="o_kanban_mobile">
                    ...
                </kanban
            </field>
        </page>
    </notebook>
    ...
```


- **CalendarView**: Les vues de calendrier affichent les enregistrements dans un calendrier quotidien, hebdomadaire ou mensuel.
```XML
    <calendar color="partner_id"
              all_day="all_day"
              date_start="date_from" date_stop="date_to"
              string="Fournisseur" mode="month"
              quick_add="False">
        <field name="partner_id"/>
    </calendar>
```

- **SearchView**:
```XML
    <search string="Search Partner">
       <field name="name"
           filter_domain="['|','|',('display_name','ilike',self),('ref','=',self),('email','ilike',self)]"/>
       <filter help="My Partners" domain="[('user_id','=',uid)]"/>
       ...
       <field name="user_id"/>
       <field name="parent_id" domain="[('is_company','=',1)]" operator="child_of"/>
       <group expand="0" name="group_by" string="Group By">
           <filter name="salesperson" string="Salesperson" domain="[]" context="{'group_by' : 'user_id'}" />
           <filter string="Company" context="{'group_by': 'parent_id'}"/>
           <filter string="Country" context="{'group_by': 'country_id'}"/>
       </group>
   </search>
```

- **Actions**:
```XML
    <record id="action_orders_to_invoice" model="ir.actions.act_window">
        <field name="name">Sales to Invoice</field>
        <field name="type">ir.actions.act_window</field>
        <field name="res_model">sale.order</field>
        <field name="view_type">form</field>
        <field name="view_mode">tree,form,calendar,graph,pivot</field>
        <field name="context">{'show_sale': True}</field>
        <field name="domain">[('invoice_status','=','to invoice')]</field>
    </record>
```


- Généralement, on définit des vues multiples pour le même modèle, à utiliser dans des situations différentes. 
```XML
    <record id="action_partner_form" model="ir.actions.act_window">
        ...
    </record>
    ...
    <record id="action_partner_form_view2" model="ir.actions.act_window.view">
        <field eval="2" name="sequence"/>
        <field name="view_mode">form</field>
        <field name="view_id" ref="view_partner_form"/>
        <field name="act_window_id" ref="action_partner_form"/>
    </record>
    <record id="action_partner_tree_view1" model="ir.actions.act_window.view">
        <field name="sequence" eval="1"/>
        <field name="view_mode">tree</field>
        <field name="view_id" ref="view_partner_tree"/>
        <field name="act_window_id" ref="action_partner_form"/>
    </record>
    <menuitem id="base.menu_sales" name="Sales" parent="base.menu_base_partner" sequence="5"/>
    <menuitem id="menu_partner_form" parent="base.menu_sales" action="action_partner_form" sequence="3"/>
```


- **Champ readonly** est un champ non modifiable par les utilisateurs de la vue:
```XML
    <field name="price" readonly="1"/>
    <field name="unit_price" attrs="{'readonly': [('state', 'in', ('confirmed', ...))]}"/>
```
```Python
    price = fields.Fload(..., readonly=True)
    ...
    name = fields.Char(..., states={'confirmed': [('readonly', True)], ...})
```
- **Onchange** permet de mettre à jour une forme chaque fois que l'utilisateur renseigne une valeur dans un champ, sans rien sauvegarder dans la base de données.
```XML
<!-- content of form view -->
<field name="amount"/>
<field name="unit_price"/>
<field name="price" readonly="1"/>
```
```Python
    ...
    @api.onchange('amount', 'unit_price')
    def _onchange_price(self):
        # set auto-changing field
        self.price = self.amount * self.unit_price
        # Can optionally return a warning and domains
        return {
            'warning': {
                'title': "Something bad happened",
                'message': "It was very bad indeed",
            }
        }
        ...
```

- **Required**: Rendre un champ obligatoire, l'utilisateur doit renseigner une valeur dans ce champ.
```XML
    <field name="unit_price" required="1"/>
    <field name="unit_price" attrs="{'required': [('state', '!=', 'draft')]}"/>
```
```Python
    unit_price = fields.Fload(..., readonly=True)
```




## L'Héritage dans Odoo

- L'héritage est un concept largement utilisé dans Odoo, il donne la flexibilité à l'ensemble du framework:

    - Héritage au niveau des modules: exprimé à travers les dépendances entre les modules.

    - Héritage au niveau des modèles: Traditional inheritance, Delegation inheritance.
    
    - Héritage au niveau des vues: Permet d'ajouter plus d'éléments à une vue existante.


### Héritage au niveau des modèles
- Héritage traditionnel: a le même mécanisme que l'héritage de programmation orienté objet:
    - ajouter des champs à un modèle ou remplacer la définition des champs sur un modèle,
    - ajouter des méthodes à un modèle ou remplacer les méthodes existantes sur un modèle.
```python
Class Model1(models.Model):
    _name = 'model.1'

    name = fields.Char(string='Nom 1', required=True,)
    ...
    @api.multi
    def action_do_something(self):
        for rec in self:
            rec.name = 'something'
    ...
Class Model2(models.Model):
    _inherit = 'model.1' # _inherit = ['model.1', 'model.3', ...]
    # _name = 'model.2' ==> New model created

    name = fields.Char(readonly=True, default='something else',)

    @api.multi
    def action_do_something(self):
        rec_with_no_names = self.filter(lambda: not rec.name)
        if rec_with_no_name:
            return super(Model2, rec_with_no_names).action_do_something()
```


- **Héritage par délégation**: permet de lier chaque enregistrement d'un modèle à un enregistrement dans un modèle parent et fournit un accès transparent aux champs de l'enregistrement parent. 

```Python
Class ResourceResource(models.Model):
    _name = 'resource.resource'
    _description = 'Resource Detail'

    name = fields.Char(required=True)
    code = fields.Char(copy=False)
    active = fields.Boolean(track_visibility='onchange', default=True, ...)
    ...
Class Employee(models.Model):
    _name = "hr.employee"
    _description = "Employee"
    _order = 'name_related'
    _inherits = {'resource.resource': "resource_id"}

    resource_id = fields.Many2one('resource.resource', string='Resource',
        ondelete='cascade', required=True, auto_join=True)
```


![Inheritance](images/inheritance_methods.png)


### Héritage au niveau des vues
```XML
    <record id="product_template_form_view" model="ir.ui.view">
        <field name="name">product.template.form.inherit</field>
        <field name="model">product.template</field>
        <field name="priority">5</field>
        <field name="inherit_id" ref="product.product_template_form_view"/>
        <field name="arch" type="xml">
            <page name="sales" position="after">
                ...
            </page>
        </field>
    </record>

    <xpath expr="//field[@name='description']" position="after">
        <field name="idea_ids" />
    </xpath>

    <field name="description" position="after">
        <field name="idea_ids" />
    </field>
```
