# Count Revisions

---

{NOTE: }

* You can get the number of revisions a document has by using the session `advanced` method `getCountFor`.

* In this page:  
   * [Get revisions count](../../../../document-extensions/revisions/client-api/session/counting#get-revisions-count)
   * [syntax](../../../../document-extensions/revisions/client-api/session/counting#syntax)

{NOTE/}

---

{PANEL: Get revisions count}

{CODE:php getCount@DocumentExtensions\Revisions\ClientAPI\Session\Counting.php /}

{PANEL/}

{PANEL: Syntax}

{CODE:php syntax@DocumentExtensions\Revisions\ClientAPI\Session\Counting.php /}

| Parameter | Type | Description |
| - | - | - |
| **id** | `string` | The ID of the document that revisions are counted for |

| Return value | |
| - | - |
| `int` | The number of revisions for the specified document |

{PANEL/}

## Related Articles

### Document Extensions

* [Document Revisions Overview](../../../../document-extensions/revisions/overview)  
* [Revert Revisions](../../../../document-extensions/revisions/revert-revisions)  
* [Revisions and Other Features](../../../../document-extensions/revisions/revisions-and-other-features)  

### Client API

* [Revisions: API Overview](../../../../document-extensions/revisions/client-api/overview)  
* [Operations: Configuring Revisions](../../../../document-extensions/revisions/client-api/operations/configure-revisions)  
* [Session: Loading Revisions](../../../../document-extensions/revisions/client-api/session/loading)  
* [Session: Including Revisions](../../../../document-extensions/revisions/client-api/session/including)  

### Studio

* [Settings: Document Revisions](../../../../studio/database/settings/document-revisions)  
* [Document Extensions: Revisions](../../../../studio/database/document-extensions/revisions)  
