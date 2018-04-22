## Odoo en profondeur (Suite)


## Rapport (Etat de sortie)

- Les etat de sortie dans Odoo sont basés sur le moteur de template **Qweb**.
- Chaque état de sortie est lié à une **action**. Lorsque l'action est appelée, le rapport est imprimé.
- Chaque état de sortie a un **format papier**. Le format papier est spécifié dans l'action.


### 1- Language QWEB.

- C'est un moteur de template XML utilisé principalement pour générer des fragments et des pages HTML.
- QWEB c'est un langage comme les autres: if, for, variable, 

```XML
<?xml version="1.0" encoding="UTF-8"?>
<templates id="template" xml:space="preserve">
...
<t t-name="TreeView">
    <select t-if="toolbar" style="width: 30%"/>
    <table class="o_treeview_table">
        <thead>
            <tr>
                <th t-foreach="fields_view" t-as="field"
                    t-if="!field.attrs.modifiers.tree_invisible"
                    class="treeview-header">
                    <t t-esc="field_value.attrs.string || fields[field.attrs.name].string" />
                </th>
            </tr>
        </thead>
        <tbody>
        </tbody>
    </table>
</t>
...
</template>
```


- **Boucle For**:

```XML
...
<tbody>
    <tr t-foreach="o.line_ids" t-as="line_ids">
        <td>
            <span t-field="line_ids.name"/>
        </td>
        ...
    </tr>
    ...
</tbody>
```

- **Condition if:**

```XML
<t t-if="o.line_ids">
    <h3>Products</h3>
    <table class="table table-condensed">
        ...
    </table>
    ...
</t>
```

```XML
<div>
    <p t-if="user.birthday == today()">Happy bithday!</p>
    <p t-elif="user.login == 'root'">Welcome master!</p>
    <p t-else="">Welcome!</p>
</div>
```

- **Affichage de données:**

```XML
...
<tbody>
    <tr t-foreach="o.order_line" t-as="line">
        <td>
            <span t-field="line.name"/>
        </td>
        <td>
            <span t-esc="', '.join(map(lambda x: x.name, line.taxes_id))"/>
        </td>
        ...
    </tr>
</tbody>
...
```


- **Définir des variables:**

```XML
...
<t t-set="foo" t-value="2 + 1"/>
<t t-esc="foo"/>
```

```XML
...
<t t-set="o" t-value="o.with_context({'lang':o.partner_id.lang})"/>
<div class="page">
...
```

- **Appel d'autres templates**

```XML
<!-- other template -->
<div>
    This template was called with content:
    <t t-raw="0"/>
</div>
<!-- inside my template -->
<t t-call="other-template">
    <em>content</em>
</t>
<!-- result -->
<div>
    This template was called with content:
    <em>content</em>
</div>
```

- **les attributs**

```XML
<t t-foreach="[1, 2, 3]" t-as="item">
    <li t-attf-class="row {{ item_parity }}"><t t-esc="item"/></li>
</t>
<!-- result -->
<li class="row even">1</li>
<li class="row odd">2</li>
<li class="row even">3</li>
```

- Pour plus de detail: https://www.odoo.com/documentation/10.0/reference/qweb.html


### 2-Structure d'un état de sortie

```XML
<template id="report_invoice">
    <t t-call="report.html_container">
        <t t-foreach="docs" t-as="o">
            <t t-call="report.external_layout">
                <div class="page">
                    <h2>Report title</h2>
                    <p>This object's name is <span t-field="o.name"/></p>
                </div>
            </t>
        </t>
    </t>
</template>
```

- Certaines variables spécifiques sont accessibles dans les états de sortie, principalement:
    1. **docs**: Enregistrements à imprimer.
    2. **time**: une référence la bibliothèque standard Python
    3. **user**: l'enregistrement `res.user` utilisateur courrant (i.e celui qui est entrain d'imprimé l'état de sortie).


- **Traduction d'un état de sortie**:

```XML
<!-- Main template -->
<template id="report_saleorder">
    <t t-call="report.html_container">
        <t t-foreach="docs" t-as="doc">
            <t t-call="sale.report_saleorder_document" t-lang="doc.partner_id.lang"/>
        </t>
    </t>
</template>
```

```XML
<!-- Translatable template -->
<template id="report_saleorder_document">
    <!-- Re-browse of the record with the partner lang -->
    <t t-set="doc" t-value="doc.with_context({'lang':doc.partner_id.lang})" />
    <t t-call="report.external_layout">
        <div class="page">
            <div class="oe_structure"/>
            <div class="row">
                <div class="col-xs-6">
                    <strong t-if="doc.partner_shipping_id == doc.partner_invoice_id">Invoice and shipping address:</strong>
                    <strong t-if="doc.partner_shipping_id != doc.partner_invoice_id">Invoice address:</strong>
                    <div t-field="doc.partner_invoice_id" t-options="{&quot;no_marker&quot;: True}"/>
                <...>
            <div class="oe_structure"/>
        </div>
    </t>
</template>
```


### astuces utiles
- **Impression de barcodes:**

```XML
<template id="report_location_barcode">
    <t t-call="web.html_container">
        <div t-foreach="[docs[x:x+4] for x in xrange(0, len(docs), 4)]" t-as="page_docs" class="page article page_stock_location_barcodes">
            <t t-foreach="page_docs" t-as="o">
                <t t-if="o.barcode"><t t-set="content" t-value="o.barcode"/></t>
                <t t-if="not o.barcode"><t t-set="content" t-value="o.name"/></t>
                <img class="barcode" t-att-src="'/report/barcode/?type=%s&amp;value=%s&amp;width=%s&amp;height=%s&amp;humanreadable=1' % ('Code128', content, 600, 100)"/>
            </t>
        </div>
    </t>
</template>
```

- **Widgets**:

```XML
<!-- Widget pour les addresses -->
<div t-if="o.dest_address_id">
    <div t-field="o.dest_address_id"
        t-options='{"widget": "contact", "fields": ["address", "name", "phone", "fax"], "no_marker": True, "phone_icons": True}'/>
<div>
```

```XML
<!-- Widget pour la devise -->
<td class="text-right">
    <span t-field="line.price_subtotal"
        t-options='{"widget": "monetary", "display_currency": o.currency_id}'/>
</td>
```

```XML
<!-- Affichage des images -->
<div class="col-xs-3">
    <img t-if="company.logo" t-att-src="'data:image/png;base64,%s' % company.logo" style="max-height: 45px;"/>
</div>
```


- **Ajout des CSS:** Trois façons différentes de le faire

1- Peut être mis directement dans la template

```XML
<span style="padding-top: 10px;" t-field="o.name"/>
```

2- En héritant la template principale des état de sortie:

```XML
<template id="report_saleorder_style" inherit_id="report.style">
    <xpath expr=".">
        <t>
          .example-css-class {
            background-color: red;
          }
        </t>
    </xpath>
</template>
```

3- En ajoutant le fichier CSS:

```XML
<template id="assets_report" inherit_id="report.assets_common">
    <xpath expr="." position="inside">
      <link href="/silog_payment_report_bordereau/static/src/less/report.less" rel="stylesheet" type="text/less"/>
    </xpath>
</template>
```


### Format de papier de l'état de sortie

```XML
<record id="paperformat_frenchcheck" model="report.paperformat">
    <field name="name">French Bank Check</field>
    <field name="default" eval="True"/>
    <field name="format">custom</field>
    <field name="page_height">80</field>
    <field name="page_width">175</field>
    <field name="orientation">Portrait</field>
    <field name="margin_top">3</field>
    <field name="margin_bottom">3</field>
    <field name="margin_left">3</field>
    <field name="margin_right">3</field>
    <field name="header_line" eval="False"/>
    <field name="header_spacing">3</field>
    <field name="dpi">80</field>
</record>
```


### Action pour impression de l'état de sortie

```XML
<report
    id="account_invoices"
    model="account.invoice"
    string="Invoices"
    report_type="qweb-pdf"
    name="account.report_invoice"
    file="account.report_invoice"
    attachment_use="True"
    attachment="(object.state in ('open','paid')) and
        ('INV'+(object.number or '').replace('/','')+'.pdf')"
/>
```

```XML
<record id="qweb_pdf_export" model="ir.actions.report.xml">
    <field name="name">Bordereau de liquidation</field>
    <field name="model">account.payment.line</field>
    <field name="type">ir.actions.report.xml</field>
    <field name="report_name">silog_payment_report_bordereau.report_silog_payment_bordereau_document</field>
    <field name="report_type">qweb-pdf</field>
    <field name="paperformat_id" ref="report.paperformat_euro"/>
    <field name="auto" eval="False"/>
    <field name="download_filename">BordLiquid_${o.gcom_file or o.name}.pdf</field>
</record>
```
- https://github.com/OCA/reporting-engine/tree/9.0/report_custom_filename


### Installation de Wkhtmltopdf

```Shell
sudo wget https://downloads.wkhtmltopdf.org/0.12/0.12.1/wkhtmltox-0.12.1_linux-trusty-amd64.deb
sudo dpkg -i wkhtmltox-0.12.1_linux-trusty-amd64.deb
sudo cp /usr/local/bin/wkhtmltopdf /usr/bin
sudo cp /usr/local/bin/wkhtmltoimage /usr/bin
```
