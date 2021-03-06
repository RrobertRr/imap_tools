.. http://docutils.sourceforge.net/docs/user/rst/quickref.html

imap_tools
==========

Working with email and mailbox using IMAP protocol.

- Parsed email message attributes
- Query builder for searching emails
- Work with emails in folders (copy, delete, flag, move, seen)
- Work with mailbox folders (list, set, get, create, exists, rename, delete, status)
- No dependencies

===============  ====================================================
Python version   3.3+
License          Apache-2.0
PyPI             https://pypi.python.org/pypi/imap_tools/
IMAP             VERSION 4rev1 - https://tools.ietf.org/html/rfc3501
===============  ====================================================

.. contents::

Installation
------------
::

    $ pip install imap_tools

Guide
-----

Basic
^^^^^
.. code-block:: python

    from imap_tools import MailBox, AND

    # get list of email subjects from INBOX folder
    with MailBox('imap.mail.com').login('test@mail.com', 'password') as mailbox:
        subjects = [msg.subject for msg in mailbox.fetch()]

    # get list of email subjects from INBOX folder - equivalent verbose version
    mailbox = MailBox('imap.mail.com')
    mailbox.login('test@mail.com', 'password', initial_folder='INBOX')  # or mailbox.folder.set instead 3d arg
    subjects = [msg.subject for msg in mailbox.fetch(AND(all=True))]
    mailbox.logout()

MailBox/MailBoxUnencrypted - for create mailbox instance.

MailBox.box - imaplib.IMAP4/IMAP4_SSL client instance.

MailBox.fetch - email message generator, first searches email ids by criteria, then fetch and yields emails by one:

* *criteria* = 'ALL' message search criteria, `docs <#search-criteria>`_
* *charset* = 'US-ASCII', indicates charset of the strings that appear in the search criteria. See rfc2978
* *limit* = None, limit on the number of read emails, useful for actions with a large number of messages, like "move"
* *miss_defect* = True, miss emails with defects
* *miss_no_uid* = True, miss emails without uid
* *mark_seen* = True, mark emails as seen on fetch
* *reverse* = False, in order from the larger date to the smaller
* *headers_only* = False, get only email headers (without text, html, attachments) !disabled until fix bug

Email attributes
^^^^^^^^^^^^^^^^

Message and Attachment public attributes are cached by functools.lru_cache

.. code-block:: python

    for msg in mailbox.fetch():
        msg.uid              # str or None: '123'
        msg.subject          # str: 'some subject 你 привет'
        msg.from_            # str: 'sender@ya.ru'
        msg.to               # tuple: ('iam@goo.ru', 'friend@ya.ru', )
        msg.cc               # tuple: ('cc@mail.ru', )
        msg.bcc              # tuple: ('bcc@mail.ru', )
        msg.reply_to         # tuple: ('reply_to@mail.ru', )
        msg.date             # datetime.datetime: 1900-1-1 for unparsed, may be naive or with tzinfo
        msg.date_str         # str: original date - 'Tue, 03 Jan 2017 22:26:59 +0500'
        msg.text             # str: 'Hello 你 Привет'
        msg.html             # str: '<b>Hello 你 Привет</b>'
        msg.flags            # tuple: ('SEEN', 'FLAGGED', 'ENCRYPTED')
        msg.headers          # dict: {'Received': ('from 1.m.ru', 'from 2.m.ru'), 'AntiVirus': ('Clean',)}

        for att in msg.attachments:  # list: [Attachment]
            att.filename         # str: 'cat.jpg'
            att.content_type     # str: 'image/jpeg'
            att.payload          # bytes: b'\xff\xd8\xff\xe0\'

        msg.obj              # email.message.Message: original object
        msg.from_values      # dict or None: {'email': 'im@ya.ru', 'name': 'Ya 你', 'full': 'Ya 你 <im@ya.ru>'}
        msg.to_values        # tuple: ({'email': '', 'name': '', 'full': ''},)
        msg.cc_values        # tuple: ({'email': '', 'name': '', 'full': ''},)
        msg.bcc_values       # tuple: ({'email': '', 'name': '', 'full': ''},)
        msg.reply_to_values  # tuple: ({'email': '', 'name': '', 'full': ''},)

Search criteria
^^^^^^^^^^^^^^^

Possible search approaches:

.. code-block:: python

    from imap_tools import AND, OR, NOT

    mailbox.fetch(AND(subject='weather'))  # query, the str-like object - see below
    mailbox.fetch('TEXT "hello"')  # str, use charset arg for non US-ASCII chars
    mailbox.fetch(b'TEXT "\xd1\x8f"')  # bytes, charset arg is ignored

Implemented query builder for search logic described in `rfc3501 <https://tools.ietf.org/html/rfc3501#section-6.4.4>`_.
See `query examples <https://github.com/ikvk/imap_tools/blob/master/examples/search.py>`_.

======  =====  ========================================== ============================================================
Class   Alias  Usage                                      Arguments
======  =====  ========================================== ============================================================
AND     A      combines keys by logical "AND" condition   Search keys (see below) | str
OR      O      combines keys by logical "OR" condition    Search keys (see below) | str
NOT     N      invert the result of a logical expression  AND/OR instances | str
Header  H      for search by headers                      name: str, value: str
======  =====  ========================================== ============================================================

If the "charset" argument is specified in MailBox.fetch, the search string will be encoded to this encoding.
You can change this behavior by overriding MailBox._criteria_encoder or pass criteria as bytes in desired encoding.

.. code-block:: python

    from imap_tools import A, AND, OR, NOT
    # AND
    A(text='hello', new=True)  # '(TEXT "hello" NEW)'
    # OR
    OR(text='hello', date=datetime.date(2000, 3, 15))  # '(OR TEXT "hello" ON 15-Mar-2000)'
    # NOT
    NOT(text='hello', new=True)  # 'NOT (TEXT "hello" NEW)'
    # complex
    A(OR(from_='from@ya.ru', text='"the text"'), NOT(OR(A(answered=False), A(new=True))), to='to@ya.ru')
    # encoding
    mailbox.fetch(A(subject='привет'), charset='utf8')  # 'привет' will be encoded by MailBox._criteria_encoder
    # python note: you can't do: A(text='two', NOT(subject='one'))
    A(NOT(subject='one'), text='two')  # use kwargs after logic classes (args)

The search key types are marked with `*` can accepts a sequence of values like list, tuple, set or generator.

=============  ==============  ======================  =================================================================
Key            Types           Results                 Description
=============  ==============  ======================  =================================================================
answered       bool            `ANSWERED|UNANSWERED`   with|without the Answered flag
seen           bool            `SEEN|UNSEEN`           with|without the Seen flag
flagged        bool            `FLAGGED|UNFLAGGED`     with|without the Flagged flag
draft          bool            `DRAFT|UNDRAFT`         with|without the Draft flag
deleted        bool            `DELETED|UNDELETED`     with|without the Deleted flag
keyword        str*            KEYWORD KEY             with the specified keyword flag
no_keyword     str*            UNKEYWORD KEY           without the specified keyword flag
`from_`        str*            FROM `"from@ya.ru"`     contain specified str in envelope struct's FROM field
to             str*            TO `"to@ya.ru"`         contain specified str in envelope struct's TO field
subject        str*            SUBJECT "hello"         contain specified str in envelope struct's SUBJECT field
body           str*            BODY "some_key"         contain specified str in body of the message
text           str*            TEXT "some_key"         contain specified str in header or body of the message
bcc            str*            BCC `"bcc@ya.ru"`       contain specified str in envelope struct's BCC field
cc             str*            CC `"cc@ya.ru"`         contain specified str in envelope struct's CC field
date           datetime.date*  ON 15-Mar-2000          internal date is within specified date
date_gte       datetime.date*  SINCE 15-Mar-2000       internal date is within or later than the specified date
date_lt        datetime.date*  BEFORE 15-Mar-2000      internal date is earlier than the specified date
sent_date      datetime.date*  SENTON 15-Mar-2000      rfc2822 Date: header is within the specified date
sent_date_gte  datetime.date*  SENTSINCE 15-Mar-2000   rfc2822 Date: header is within or later than the specified date
sent_date_lt   datetime.date*  SENTBEFORE 1-Mar-2000   rfc2822 Date: header is earlier than the specified date
size_gt        int >= 0        LARGER 1024             rfc2822 size larger than specified number of octets
size_lt        int >= 0        SMALLER 512             rfc2822 size smaller than specified number of octets
new            True            NEW                     have the Recent flag set but not the Seen flag
old            True            OLD                     do not have the Recent flag set
recent         True            RECENT                  have the Recent flag set
all            True            ALL                     all, criteria by default
uid            iter(str)|str   UID 1,2,17              corresponding to the specified unique identifier set
header         H(str, str)*    HEADER "A-Spam" "5.8"   have a header that contains the specified str in the text
gmail_label    str*            X-GM-LABELS "some_lbl"  have this gmail label.
=============  ==============  ======================  =================================================================

Server side search notes:

* For string search keys a message matches if the string is a substring of the field. The matching is case-insensitive.
* When searching by dates - email's time and timezone are disregarding.

Actions with emails in folder
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can use 2 approaches to perform these operations:

* "in bulk" - Perform IMAP operation for message set per 1 command
* "by one" - Perform IMAP operation for each message separately per N commands

Result of MailBox.fetch generator in actions will be implicitly converted to uid list.

For actions with a large number of messages imap command may be too large and will throw an exception,
use 'limit' argument for fetch in this case.

.. code-block:: python

    with MailBox('imap.mail.com').login('test@mail.com', 'pwd', initial_folder='INBOX') as mailbox:

        # COPY all messages from current folder to folder1, *by one
        for msg in mailbox.fetch():
            res = mailbox.copy(msg.uid, 'INBOX/folder1')

        # MOVE all messages from current folder to folder2, *in bulk (implicit creation of uid list)
        mailbox.move(mailbox.fetch(), 'INBOX/folder2')

        # DELETE all messages from current folder, *in bulk (explicit creation of uid list)
        mailbox.delete([msg.uid for msg in mailbox.fetch()])

        # FLAG unseen messages in current folder as Answered and Flagged, *in bulk.
        flags = (imap_tools.MailMessageFlags.ANSWERED, imap_tools.MailMessageFlags.FLAGGED)
        mailbox.flag(mailbox.fetch('(UNSEEN)'), flags, True)

        # SEEN: mark all messages sent at 05.03.2007 in current folder as unseen, *in bulk
        mailbox.seen(mailbox.fetch("SENTON 05-Mar-2007"), False)

Actions with mailbox folders
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

    with MailBox('imap.mail.com').login('test@mail.com', 'pwd') as mailbox:
        # LIST
        for folder_info in mailbox.folder.list('INBOX'):
            print(folder_info)  # {'name': 'INBOX|cats', 'delim': '|', 'flags': ('\\Unmarked', '\\HasChildren')}
        # SET
        mailbox.folder.set('INBOX')
        # GET
        current_folder = mailbox.folder.get()
        # CREATE
        mailbox.folder.create('folder1')
        # EXISTS
        is_exists = mailbox.folder.exists('folder1')
        # RENAME
        mailbox.folder.rename('folder1', 'folder2')
        # DELETE
        mailbox.folder.delete('folder2')
        # STATUS
        folder_status = mailbox.folder.status('some_folder')
        print(folder_status)  # {'MESSAGES': 41, 'RECENT': 0, 'UIDNEXT': 11996, 'UIDVALIDITY': 1, 'UNSEEN': 5}

Exceptions
^^^^^^^^^^

Custom lib exceptions here: `errors.py <https://github.com/ikvk/imap_tools/blob/master/imap_tools/errors.py>`_.

Reasons
-------

- Excessive low level of `imaplib` library.
- Other libraries contain various shortcomings or not convenient.
- Open source projects makes world better.

Release notes
-------------
 `release_notes.rst <https://github.com/ikvk/imap_tools/blob/master/release_notes.rst>`_

Contribute
----------

If you found a bug or have a question, please let me know - create merge request or issue.

Thanks to:

`shilkazx <https://github.com/shilkazx>`_,
`somepad <https://github.com/somepad>`_,
`0xThiebaut <https://github.com/0xThiebaut>`_,
`TpyoKnig <https://github.com/TpyoKnig>`_,
`parchd-1 <https://github.com/parchd-1>`_,
`dojasoncom <https://github.com/dojasoncom>`_,
`RandomStrangerOnTheInternet <https://github.com/RandomStrangerOnTheInternet>`_,
`jonnyarnold <https://github.com/jonnyarnold>`_,
`Mitrich3000 <https://github.com/Mitrich3000>`_,
`audemed44 <https://github.com/audemed44>`_,
`mkalioby <https://github.com/mkalioby>`_,
`atlas0fd00m <https://github.com/atlas0fd00m>`_,
`unqx <https://github.com/unqx>`_,
`daitangio <https://github.com/daitangio>`_,
`upils <https://github.com/upils>`_,
`Foosec <https://github.com/Foosec>`_,
`frispete <https://github.com/frispete>`_,
`PH89 <https://github.com/PH89>`_
