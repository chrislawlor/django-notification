=====
Usage
=====

Integrating notification support into your app is a simple three-step process.

  * create your notice types
  * create your notice templates
  * send notifications

Creating Notice Types
=====================

You need to call ``NoticeType.create(label, display, description)`` once to
create the notice types for your application in the database. ``label`` is just
the internal shortname that will be used for the type, ``display`` is what the
user will see as the name of the notification type and `description` is a
short description.

For example::

    notification.create_notice_type("friends_invite", "Invitation Received", "you have received an invitation")

One good way to automatically do this notice type creation is in a
``management.py`` file for your app, attached to the syncdb signal.
Here is an example::

    from django.conf import settings
    from django.utils.translation import ugettext_noop as _
    
    if "notification" in settings.INSTALLED_APPS:
        from notification.models import NoticeType
        
        def create_notice_types(app, created_models, verbosity, **kwargs):
            NoticeType.create("friends_invite", _("Invitation Received"), _("you have received an invitation"))
            NoticeType.create("friends_accept", _("Acceptance Received"), _("an invitation you sent has been accepted"))
            
        signals.post_syncdb.connect(create_notice_types, sender=notification)
    else:
        print "Skipping creation of NoticeTypes as notification app not found"


Notice that the code is wrapped in a conditional clause so if
django-notification is not installed, your app will proceed anyway.

Note that the display and description arguments are marked for translation by
using ugettext_noop. That will enable you to use Django's makemessages
management command and use django-notification's i18n capabilities.

Notification templates
======================

There are six different templates that can to be written for the actual content of the notices:

  * ``short.txt`` is a very short, text-only version of the notice.
  * ``full.txt`` is a longer, text-only version of the notice.
  * ``email_body.txt`` is used to render the body of a notification email, usually including a link back to the site.
  * ``email_subject.txt``  is used to render the subject of a notification email.
  * ``notice.html`` is a short, html version of the notice, displayed in a user's notice list on the website
  * ``full.html`` is a long, html version of the notice (not currently used for anything)

Each of these should be put in a directory on the template path called ``notification/<notice_type_label>/<template_name>``.
If any of these are missing, a default would be used. In practice, ``notice.html`` and ``full.txt`` should be provided at a minimum.

For example, ``notification/friends_invite/notice.html`` might contain::
    
    {% load i18n %}{% url invitations as invitation_page %}{% url profile_detail username=invitation.from_user.username as user_url %}
    {% blocktrans with invitation.from_user as invitation_from_user %}<a href="{{ user_url }}">{{ invitation_from_user }}</a> has requested to add you as a friend (see <a href="{{ invitation_page }}">invitations</a>){% endblocktrans %}

and ``notification/friends_full.txt`` might contain::
    
    {% load i18n %}{% url invitations as invitation_page %}{% blocktrans with invitation.from_user as invitation_from_user %}{{ invitation_from_user }} has requested to add you as a friend. You can accept their invitation at:
    
    http://{{ current_site }}{{ invitation_page }}
    {% endblocktrans %}

The context variables are provided when sending the notification.


Sending Notification
====================

There are two different ways of sending out notifications. We have support
for blocking and non-blocking methods of sending notifications. The most
simple way to send out a notification, for example::

    notification.send([to_user], "friends_invite", {"from_user": from_user})

One thing to note is that ``send`` is a proxy around either ``send_now`` or
``queue``. They all have the same signature::

    send(users, label, extra_context)

The parameters are:

 * ``users`` is an iterable of ``User`` objects to send the notification to.
 * ``label`` is the label you used in the previous step to identify the notice
   type.
 * ``extra_content`` is a dictionary to add custom context entries to the
   template used to render to notification. This is optional.

``send_now`` vs. ``queue`` vs. ``send``
---------------------------------------

Lets first break down what each does.

``send_now``
~~~~~~~~~~~~

This is a blocking call that will check each user for elgibility of the
notice and actually peform the send.

``queue``
~~~~~~~~~

This is a non-blocking call that will queue the call to ``send_now`` to
be executed at a later time. To later execute the call you need to use
the ``emit_notices`` management command.

``send``
~~~~~~~~

A proxy around ``send_now`` and ``queue``. It gets its behavior from a global
setting named ``NOTIFICATION_QUEUE_ALL``. By default it is ``False``. This
setting is meant to help control whether you want to queue any call to
``send``.

``send`` also accepts ``now`` and ``queue`` keyword arguments. By default
each option is set to ``False`` to honor the global setting which is ``False``.
This enables you to override on a per call basis whether it should call
``send_now`` or ``queue``.

Optional notification support
-----------------------------

In case you want to use django-notification in your reusable app, you can
wrap the import of django-notification in a conditional clause that tests
if it's installed before sending a notice. As a result your app or
project still functions without notification.

For example::

    from django.conf import settings

    if "notification" in settings.INSTALLED_APPS:
        from notification import models as notification
    else:
        notification = None

and then, later::

    if notification:
        notification.send([to_user], "friends_invite", {"from_user": from_user})
