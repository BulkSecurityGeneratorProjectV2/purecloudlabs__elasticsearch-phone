# Elasticsearch-Phone

Indexing phone numbers & sip addresses in lucene is complicated. Most people use ngram tokenizers. We did that for a while with ngram min=3 & max=35, but the result was often 100s of tokens per sip address. Working in a call center focused company we quickly figured out how wasteful that is on the storage front. For us 6/7ths of our indexes were waisted on useless sip address tokens.

It's a hard problem to regex your way out of. An international phone number often includes a country code, but that can be 1, 2, or 3+ digits. A lot of people have requested elasticsearch integrate google's libphone library into a custom lucene analyzer. It hasn't happened yet, so here's a plugin that attempts to do just that.

Note: This is a young project. We'll improve as time goes on, but use at your own risk.

# Building and installing the plugin
mvn package
./bin/plugin --url file:///....elasticsearch-phone/target/releases/elasticsearch-phone-1.0.0.zip --install elasticsearch-phone;

# Analyzers

This project provides three analyzers that are intended for different contexts.

* The `phone` analyzer supports SIP URIs and other phone numbers and is intended to be used when indexing. It strips common prefixes such as `sip:` and `tel:` (and indexes those as separate tokens) and tokenizes the phone number with various prefix lengths.
* The `phone-email` analyzer extends the `phone` analyzer with additional tokenization for email addresses (e.g. generating tokens for the user part and the domain part of an email address).
* The `phone-search` analyzer is intended to be used as a `search_analyzer` with one of the other two analyzers used for indexing. It does minimal tokenization: If a term starts with `sip:` or `tel:` it strips this part and generates a token for it. The analyzer also strips a leading `+` from phone numbers.

All three analyzers remove non-unique tokens and transform terms to lowercase.


## Example inputs

Provide a telephone or sip address prefixed by `tel:` or `sip:` with no spaces or symbols.

Your indexing template will need to specify the analyzer for the field. EG
```json
            "field": {
              "type": "string",
              "analyzer": "phone",
              "search_analyzer": "phone-search"
            }
```

Sample allowed inputs (see **PhoneTokenizerIntegrationTest** and **PhoneSearchIntegrationTest** for more):
* tel:+441344840400
* tel:+498362930830
* sip:abc@autosbcpc
* sip:+13119310462;ext=2244@178.12.10.115:8060

## Example tokenization

### SIP URI
Input (with country code): `sip:+13169410766;ext=2233@172.17.10.117:8060`

Tokens:

```
sip:+13169410766;ext=2233@172.17.10.117:8060
sip:
13169410766;ext=2233@172.17.10.117:8060
13169410766;ext=2233
1
2233
3169410766
3
13
31
131
316
1316
3169
13169
31694
131694
316941
1316941
3169410
13169410
31694107
131694107
316941076
1316941076
13169410766
```

### Phone number
Input (without a country code): `tel:8177148350`

Tokens:

```
tel:8177148350
tel:
8177148350
8
81
817
8177
81771
817714
8177148
81771483
817714835
```

### Email address
Input: `user.name@domain.com`

Tokens:

```
user.name@domain.com
user.name
user *
name *
domain.com *
domain *
com *
```

Tokens marked with `*` are only generated by the `phone-email` tokenizer.

## Search examples

### Term
Term queries will return exact matches without analyzing (without normalization as lowercase).
```json
"query": {
  "term" : { "field" : "8177" }
}
```

```json
"query": {
  "term" : { "field" : "domain" }
}
```

### Match
Match queries use the configured `analyzer` (or `search_analyzer`). In this example, the query will be translated to a boolean `and` of two term queries for (`tel:` and `8177`).
```json
"query": {
  "match" : {
      "field" : {
          "query" : "tel:8177",
          "operator" : "and"
      }
  }
}
```
