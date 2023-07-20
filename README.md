[![PyPi Release](https://img.shields.io/pypi/v/rcsbsearchapi.svg)](https://pypi.org/project/rcsbsearchapi/)
[![Build Status](https://dev.azure.com/rcsb/RCSB%20PDB%20Python%20Projects/_apis/build/status/rcsb.py-rcsbsearchapi?branchName=master)](https://dev.azure.com/rcsb/RCSB%20PDB%20Python%20Projects/_build/latest?definitionId=38&branchName=master)
[![Documentation Status](https://readthedocs.org/projects/rcsbsearchapi/badge/?version=latest)](https://rcsbsearchapi.readthedocs.io/en/latest/?badge=latest)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/rcsb/py-rcsbsearchapi/master)

# rcsbsearchapi

Python interface for the RCSB PDB Search API.

Currently the 'text search' part of the API has been implemented. See 'Supported
features' below.

This package requires python 3.7 or later.

## Example

Here is a quick example of how the package is used. Two syntaxes are available for
constructing queries: an "operator" API using python's comparators, and a "fluent"
syntax where terms are chained together. Which to use is a matter of preference.
Below, you will find examples of Full text and attribute search and sequence search.

A runnable jupyter notebook with this example is available in [notebooks/quickstart.ipynb](notebooks/quickstart.ipynb), or can be run online using binder:
[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/rcsb/py-rcsbsearchapi/master?labpath=notebooks%2Fquickstart.ipynb)

An additional example including a Covid-19 related example is in [notebooks/covid.ipynb](notebooks/covid.ipynb):
[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/rcsb/py-rcsbsearchapi/master?labpath=notebooks%2Fcovid.ipynb)

### Operator Example

Here is an example from the [RCSB PDB Search
API](http://search.rcsb.org/#search-example-1) page, using the operator syntax. This
query finds symmetric dimers having a twofold rotation with the DNA-binding domain of
a heat-shock transcription factor. Full text and attribute search is used.
```python
from rcsbsearchapi.search import TextQuery
from rcsbsearchapi import rcsb_attributes as attrs

# Create terminals for each query
q1 = TextQuery('"heat-shock transcription factor"')
q2 = attrs.rcsb_struct_symmetry.symbol == "C2"
q3 = attrs.rcsb_struct_symmetry.kind == "Global Symmetry"
q4 = attrs.rcsb_entry_info.polymer_entity_count_DNA >= 1

# combined using bitwise operators (&, |, ~, etc)
query = q1 & (q2 & q3 & q4)

# Call the query to execute it
for assemblyid in query("assembly"):
    print(assemblyid)
```
For a full list of attributes, please refer to the [RCSB PDB
schema](http://search.rcsb.org/rcsbsearch/v2/metadata/schema).

### Fluent Example

Here is the same example using the
[fluent](https://en.wikipedia.org/wiki/Fluent_interface) syntax.
```python
from rcsbsearchapi.search import TextQuery, AttributeQuery, Attr

# Start with a Attr or TextQuery, then add terms
results = TextQuery("heat-shock transcription factor").and_(
    # Add attribute node as fully-formed AttributeQuery
    AttributeQuery("rcsb_struct_symmetry.symbol", operator="exact_match", "C2") \
    # Add attribute node as Attr with chained operations
    .and_(Attr("rcsb_struct_symmetry.kind")).exact_match("Global Symmetry") \
    # Add attribute node by name (converted to Attr) with chained operations
    .and_("rcsb_entry_info.polymer_entity_count_DNA").greater_or_equal(1)
    ).exec("assembly")

# Exec produces an iterator of IDs
for assemblyid in results:
    print(assemblyid)
```
### Structural Attribute Search and Chemical Attribute Search Combination

Grouping of a Structural Attribute query and Chemical Attribute query is permitted as long as grouping is done correctly and search services are specified accordingly. Note the example below. More details on attributes that are available for text searches can be found on the [RCSB PDB Search API](https://search.rcsb.org/#search-attributes) page.
```python
from rcsbsearchapi.const import CHEMICAL_ATTRIBUTE_SEARCH_SERVICE, STRUCTURE_ATTRIBUTE_SEARCH_SERVICE
from rcsbsearchapi.search import AttributeQuery

# By default, service is set to "text" for structural attribute search
q1 = AttributeQuery("exptl.method", "exact_match", "electron microscopy",
                    STRUCTURE_ATTRIBUTE_SEARCH_SERVICE # this constant specifies "text" service
                    )

# Need to specify chemical attribute search service - "text_chem"
q2 = AttributeQuery("drugbank_info.brand_names", "contains_phrase", "tylenol",
                    CHEMICAL_ATTRIBUTE_SEARCH_SERVICE # this constant specifies "text_chem" service
                    )

query = q1 & q2 # combining queries

list(query())
```
### Computed Structure Models

The [RCSB PDB Search API](https://search.rcsb.org/#results_content_type)
page provides information on how to include Computed Models into a search query. Here is a code example below.
This query returns ID's for experimental and computed models associated with "hemoglobin". 
Queries with only computed models or only experimental models can be made.
```python
from rcsbsearchapi.search import TextQuery

q1 = TextQuery("hemoglobin")

# add parameter as a list with either "computational" or "experimental" or both as list values
q2 = q1(return_content_type=["computational", "experimental"])

list(q2)
```
### Return Types and Attribute Search

A search query can return different result types when a return type is specified. 
Below are examples on specifying return types Polymer Entities,
Non-polymer Entities, Polymer Instances, and Molecular Definitions, using a Structure Attribute query. 
More information on return types can be found in the 
[RCSB PDB Search API](https://search.rcsb.org/#building-search-request) page.
```python
from rcsbsearchapi.search import AttributeQuery

q1 = AttributeQuery("rcsb_entry_container_identifiers.entry_id", "in", ["4HHB"]) # query for 4HHB deoxyhemoglobin

# Polymer entities
for poly in q1("polymer_entity"): # include return type as a string parameter for query object
    print(poly)
    
# Non-polymer entities
for nonPoly in q1("non_polymer_entity"):
    print(nonPoly)
    
# Polymer instances
for polyInst in q1("polymer_instance"):
    print(polyInst)
    
# Molecular definitions
for mol in q1("mol_definition"):
    print(mol)
```
### Protein Sequence Search Example

Below is an example from the [RCSB PDB Search API](https://search.rcsb.org/#search-example-3) page,
using the sequence search function.
This query finds macromolecular PDB entities that share 90% sequence identity with
GTPase HRas protein from *Gallus gallus* (*Chicken*).
```python
from rcsbsearchapi.search import SequenceQuery

# Use SequenceQuery class and add parameters
results = SequenceQuery("MTEYKLVVVGAGGVGKSALTIQLIQNHFVDEYDPTIEDSYRKQVVIDGET" +
                        "CLLDILDTAGQEEYSAMRDQYMRTGEGFLCVFAINNTKSFEDIHQYREQI" +
                        "KRVKDSDDVPMVLVGNKCDLPARTVETRQAQDLARSYGIPYIETSAKTRQ" +
                        "GVEDAFYTLVREIRQHKLRKLNPPDESGPGCMNCKCVIS", 1, 0.9)
    
# results("polymer_entity") produces an iterator of IDs with return type - polymer entities
for polyid in results("polymer_entity"):
    print(polyid)
```
### Sequence Motif Search Example

Below is an example from the [RCSB PDB Search API](https://search.rcsb.org/#search-example-6) page,
using the sequence motif search function. 
This query retrives occurences of the His2/Cys2 Zinc Finger DNA-binding domain as
represented by its PROSITE signature. 
```python
from rcsbsearchapi.search import SeqMotifQuery

# Use SeqMotifQuery class and add parameters
results = SeqMotifQuery("C-x(2,4)-C-x(3)-[LIVMFYWC]-x(8)-H-x(3,5)-H.",
                        pattern_type="prosite",
                        sequence_type="protein")

# results("polymer_entity") produces an iterator of IDs with return type - polymer entities
for polyid in results("polymer_entity"):
    print(polyid)
```

You can also use a regular expression (RegEx) to make a sequence motif search.
As an example, here is a query for the zinc finger motif that binds Zn in a DNA-binding domain:
```python
from rcsbsearchapi.search import SeqMotifQuery

results = SeqMotifQuery("C.{2,4}C.{12}H.{3,5}H", pattern_type="regex", sequence_type="protein")

for polyid in results("polymer_entity"):
    print(polyid)
```

You can use a standard amino acid sequence to make a sequence motif search. 
X can be used to allow any amino acid in that position. 
As an example, here is a query for SH3 domains:
```python
from rcsbsearchapi.search import SeqMotifQuery

# By default, the pattern_type argument is "simple" and the sequence_type argument is "protein".
results = SeqMotifQuery("XPPXP")  # X is used as a "variable residue" and can be any amino acid. 

for polyid in results("polymer_entity"):
    print(polyid)
```

All 3 of these pattern types can be used to search for DNA and RNA sequences as well.
Demonstrated are 2 queries, one DNA and one RNA, using the simple pattern type:
```python
from rcsbsearchapi.search import SeqMotifQuery

# DNA query: this is a query for a T-Box.
dna = SeqMotifQuery("TCACACCT", sequence_type="dna")

print("DNA results:")
for polyid in dna("polymer_entity"):
    print(polyid)

# RNA query: 6C RNA motif
rna = SeqMotifQuery("CCCCCC", sequence_type="rna")
print("RNA results:")
for polyid in rna("polymer_entity"):
    print(polyid)
```
## Supported Features

The following table lists the status of current and planned features.

- [x] Structure and chemical attribute search
  - [x] Attribute Comparison operations
  - [x] Query set operations
  - [x] Attribute `contains`, `in_` (fluent only)
- [x] Option to include computed structure models (CSMs) in search
- [x] Sequence search
- [x] Sequence motif search
- [ ] Structural search
- [ ] Structural motif search
- [ ] Chemical search
- [ ] Rich results using the Data API

Contributions are welcome for unchecked items!

## Installation

Get it from pypi:

    pip install rcsbsearchapi

Or, download from [github](https://github.com/rcsb/py-rcsbsearchapi)

## Documentation

Detailed documentation is at [rcsbsearchapi.readthedocs.io](https://rcsbsearchapi.readthedocs.io/en/latest/)

## License

Code is licensed under the BSD 3-clause license. See [LICENSE](LICENSE) for details.

## Citing rcsbsearchapi

Please cite the rcsbsearchapi package by URL:

> https://rcsbsearchapi.readthedocs.io

You should also cite the RCSB PDB service this package utilizes:

> Yana Rose, Jose M. Duarte, Robert Lowe, Joan Segura, Chunxiao Bi, Charmi
> Bhikadiya, Li Chen, Alexander S. Rose, Sebastian Bittrich, Stephen K. Burley,
> John D. Westbrook. RCSB Protein Data Bank: Architectural Advances Towards
> Integrated Searching and Efficient Access to Macromolecular Structure Data
> from the PDB Archive, Journal of Molecular Biology, 2020.
> DOI: [10.1016/j.jmb.2020.11.003](https://doi.org/10.1016/j.jmb.2020.11.003)

## Attributions

The source code for this project was originally written by [Spencer Bliven](https://github.com/sbliven) and forked
from https://github.com/sbliven/rcsbsearch. We would like to express our tremendous
gratitude for his generous efforts in designing such a comprehensive public utility
Python package for interacting with the RCSB PDB search API, [rcsbsearch](https://rcsbsearchapi.readthedocs.io).

## Developers

For information about building and developing `rcsbsearchapi`, see
[CONTRIBUTING.md](CONTRIBUTING.md)
