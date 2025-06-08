# python-gst-invoice-generator

!pip install fpdf num2words

import os
from fpdf import FPDF
from google.colab import files
from datetime import datetime
from math import ceil

# Ensure output directory exists
os.makedirs('/mnt/data', exist_ok=True)

def get_item_details(item_name):
    print(f"\nEnter details for {item_name}:")
    qty = int(input("Quantity: "))
    rate_incl_gst = float(input("Rate per unit (Rs.) [Including GST]: "))
    gst_percent = float(input("GST %: "))
    
    rate_taxable = rate_incl_gst / (1 + gst_percent / 100)
    return qty, rate_incl_gst, gst_percent, rate_taxable

client_name = input("Enter Client Name: ")

items_input = [
    "250ml PET Bottle Premium",
    "250ml PET Bottle Premium Plus",
    "1L PET Bottle Premium",
    "1L PET Bottle Premium Plus",
    "Labelling Charges"
]

items = []
for item in items_input:
    qty, rate_incl_gst, gst_percent, rate_taxable = get_item_details(item)
    items.append((item, qty, rate_incl_gst, gst_percent, rate_taxable))

total_taxable = 0
total_cgst = 0
total_sgst = 0

calculated_items = []

for desc, qty, rate_incl_gst, gst_percent, rate_taxable in items:
    taxable = qty * rate_taxable
    cgst = round(taxable * (gst_percent / 2) / 100, 2)
    sgst = round(taxable * (gst_percent / 2) / 100, 2)
    total_taxable += taxable
    total_cgst += cgst
    total_sgst += sgst
    hsn = "22011020" if "Bottle" in desc else "998591"
    calculated_items.append((desc, hsn, qty, rate_taxable, taxable, gst_percent, cgst, sgst))

total_gst = total_cgst + total_sgst
invoice_total = total_taxable + total_gst
invoice_total_rounded = ceil(invoice_total)

invoice_date = datetime.now()
invoice_no = f"AAM/GST/{invoice_date.strftime('%Y%m%d')}/001"

class GSTInvoicePDF(FPDF):
    def header(self):
        self.set_font("Arial", "B", 14)
        self.cell(0, 10, "AAMARAM", ln=1, align="C")
        self.set_font("Arial", "", 9)
        self.cell(0, 5, "Manufacturer & Supplier of Water Bottle Packaging Solutions", ln=1, align="C")
        self.cell(0, 5, "GSTIN: 03NC*******F1*K", ln=1, align="C")
        self.cell(0, 5, "Phone: +91 7@@@1 @@@@4 | Email: aamaram564@gmail.com", ln=1, align="C")
        self.ln(3)
        self.set_font("Arial", "B", 12)
        self.cell(0, 8, "TAX INVOICE", ln=1, align="C")
        self.ln(2)

    def add_invoice_details(self):
        self.set_font("Arial", "", 8)
        self.cell(120, 6, f"Invoice No: {invoice_no}", border=0)
        self.cell(0, 6, f"Date: {invoice_date.strftime('%d/%m/%Y')}", ln=1, border=0)
        self.ln(1)

        self.set_font("Arial", "B", 8)
        self.cell(0, 6, "Buyer Details:", ln=1)

        self.set_font("Arial", "", 8)
        buyer_info = f"""Name: {client_name}
Billing Address:
Shipping Address:
GSTIN : 03NC*******F1*K
City: Ludhiana
State: Punjab | PIN 141001"""
        self.multi_cell(0, 5, buyer_info)
        self.ln(3)

    def add_goods_table(self):
        self.set_font("Arial", "B", 8)
        headers = [
            ("Sr", 8, 'C'),
            ("Description", 48, 'L'),
            ("HSN", 18, 'C'),
            ("Qty", 14, 'C'),
            ("Rate", 20, 'R'),
            ("Taxable", 20, 'R'),
            ("GST %", 12, 'C'),
            ("CGST", 16, 'R'),
            ("SGST", 16, 'R'),
            ("Total", 15, 'R')
        ]

        for header, width, align in headers:
            self.cell(width, 7, header, border=1, align=align)
        self.ln()

        self.set_font("Arial", "", 8)
        for i, (desc, hsn, qty, rate_taxable, taxable, gst_percent, cgst, sgst) in enumerate(calculated_items, 1):
            total = taxable + cgst + sgst
            row = [
                (str(i), 8, 'C'),
                (desc, 48, 'L'),
                (hsn, 18, 'C'),
                (str(qty), 14, 'C'),
                (f"{rate_taxable:.2f}", 20, 'R'),
                (f"{taxable:.2f}", 20, 'R'),
                (f"{gst_percent:.2f}", 12, 'C'),
                (f"{cgst:.2f}", 16, 'R'),
                (f"{sgst:.2f}", 16, 'R'),
                (f"{total:.2f}", 15, 'R')
            ]
            for text, width, align in row:
                self.cell(width, 7, text, border=1, align=align)
            self.ln()

        self.set_font("Arial", "B", 8)
        self.cell(131, 7, "Total Taxable Value", border=1, align='R')
        self.cell(61, 7, f"{total_taxable:.2f}", border=1, align='R')
        self.ln(8)

    def add_tax_summary(self):
        self.set_font("Arial", "B", 9)
        self.cell(0, 7, "GST Summary", ln=1)

        self.set_font("Arial", "B", 8)
        self.cell(60, 7, "Taxable Amount", border=1, align='C')
        self.cell(60, 7, "CGST Amount", border=1, align='C')
        self.cell(60, 7, "SGST Amount", border=1, align='C')
        self.ln()

        self.set_font("Arial", "", 8)
        self.cell(60, 7, f"{total_taxable:.2f}", border=1, align='R')
        self.cell(60, 7, f"{total_cgst:.2f}", border=1, align='R')
        self.cell(60, 7, f"{total_sgst:.2f}", border=1, align='R')
        self.ln(10)

    def add_total_amount(self):
        self.set_font("Arial", "B", 10)
        self.cell(0, 8, "Total Invoice Value", ln=1)

        self.set_font("Arial", "", 8)
        self.cell(120, 7, "Taxable Amount", border=1, align='R')
        self.cell(70, 7, f"{total_taxable:.2f}", border=1, align='R')
        self.ln()
        self.cell(120, 7, "Add: GST", border=1, align='R')
        self.cell(70, 7, f"{total_gst:.2f}", border=1, align='R')
        self.ln()
        self.cell(120, 7, "Invoice Total (Rounded Up)", border=1, align='R')
        self.cell(70, 7, f"{invoice_total_rounded:.2f}", border=1, align='R')
        self.ln(10)

    def add_footer(self):
        self.set_font("Arial", "", 8)
        self.multi_cell(0, 6, "Declaration: We declare that this invoice shows the actual price of the goods described and that all particulars are true and correct.")
        self.ln(6)
        self.cell(0, 6, "Authorized Signatory", ln=1)
        self.cell(0, 6, "(For AAMARAM)", ln=1)
        self.cell(0, 10, "Signature: _______________", ln=1)
        self.cell(0, 6, "Name: ", ln=1)
        self.cell(0, 6, "Designation: Proprietor / Director / Manager", ln=1)

pdf = GSTInvoicePDF(orientation='P', unit='mm', format='A4')  
pdf.add_page()
pdf.add_invoice_details()
pdf.add_goods_table()
pdf.add_tax_summary()
pdf.add_total_amount()
pdf.add_footer()

filename = f"/mnt/data/Invoice_{client_name.replace(' ', '_')}_{invoice_date.strftime('%d%m%Y')}.pdf"
pdf.output(filename)

print(f"Invoice saved as {filename}")

files.download(filename)
