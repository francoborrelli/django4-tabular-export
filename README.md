  # django4-tabular-export

  [![PyPI version](https://badge.fury.io/py/django4-tabular-export.svg)](https://badge.fury.io/py/django4-tabular-export)

> [!NOTE]  
> This version is compatible with Django 4+.

## Documentation

This module contains functions which take (headers, rows) pairs and
return HttpResponses with either XLSX or CSV downloads and Django admin
actions which can be added to any ModelAdmin for generic exports. It
provides two functions (`export_to_csv_response` and
`export_to_xlsx_response`) which take a filename, a list of column
headers, and a Django `QuerySet`, list-like object, or generator and
return a response.

## Goals

-   This project is not intended to be a general-purpose spreadsheet
    manipulation library. The only goal is to export data quickly and
    safely.
-   The API is intentionally simple, giving you full control over the
    display and formatting of headers or your data. `flatten_queryset`
    has special handling for only two types of data: `None` will be
    converted to an empty string and `date` or `datetime` instances will
    serialized using `isoformat()`. All other values will be specified
    as the text data type to avoid data corruption in Excel if the
    values happen to resemble a date in the current locale.
-   **Unicode-safety**: input values, including lazy objects, are
    converted using Django\'s
    [force_str](https://docs.djangoproject.com/en/5.0/ref/utils/#django.utils.encoding.force_str)
    function and will always be emitted as UTF-8
-   **Performance**: the code is known to work with data sets up to
    hundreds of thousands of rows. CSV responses use
    `StreamingHttpResponse`, use minimal memory, and start very quickly.
    Excel (XLSX) responses cannot be streamed but
    [xlsxwriter](https://pypi.python.org/pypi/XlsxWriter) is one of the
    faster implementations and its memory-size optimizations are
    enabled.

# Quickstart

Install django4-tabular-export:

```py
pip install django4-tabular-export
```

Then use it in a project:
```py
from tabular_export.core import export_to_csv_response, export_to_excel_response, flatten_queryset

def my_view(request):
    return export_to_csv_response('test.csv', ['Column 1'], [['Data 1'], ['Data 2']])


def my_other_view(request):
    headers = ['Title', 'Date Created']
    rows = MyModel.objects.values_list('title', 'date_created')
    return export_to_excel_response('items.xlsx', headers, rows)


def export_using_a_generator(request):
    headers = ['A Number']

def my_generator():
    for i in range(0, 100000):
        yield (i, )

  return export_to_excel_response('numbers.xlsx', headers, my_generator())

def export_renaming_columns(request):
    qs = MyModel.objects.filter(foo="…").select_related("…")
    headers, data = flatten_queryset(qs, field_names=['title', 'related_model__title_en'], extra_verbose_names={'related_model__title_en': 'English Title'})
    return export_to_csv_response('custom_export.csv', headers, data)
```

## Admin Integration

There are two convenience [admin
actions](https://docs.djangoproject.com/en/5.0/ref/contrib/admin/actions/)
which make it simple to add "Export to Excel" and "Export to CSV"
actions:

```py
from tabular_export.admin import export_to_csv_action, export_to_excel_action

class MyModelAdmin(admin.ModelAdmin):
    actions = (export_to_excel_action, export_to_csv_action)
```

The default columns will be the same as you would get calling
`values_list` on your `ModelAdmin`\'s default queryset as returned by
`ModelAdmin.get_queryset()`. If you want to customize this, simply
declare a new action on your `ModelAdmin` which does whatever data
preparation is necessary:

```py
from tabular_export.admin import export_to_excel_action

class MyModelAdmin(admin.ModelAdmin):
    actions = ('export_batch_summary_action', )

def export_batch_summary_action(self, request, queryset):
    headers = ['Batch Name', 'My Computed Field']
    rows = queryset.annotate("…").values_list('title', 'computed_field_name')
    return export_to_excel_response('batch-summary.xlsx', headers, rows)
    export_batch_summary_action.short_description = 'Export Batch Summary'
```

## Debugging

The `TABULAR_RESPONSE_DEBUG = True` setting will cause all views to
return HTML tables
