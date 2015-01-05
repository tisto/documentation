=============================
 Site setup and configuration
=============================

.. admonition:: Description

    How to create settings for your add-on product and how to
    programmatically add new Plone control panel entries.

.. contents:: :local:

Introduction
------------

Sometimes when you create an add-on product for Plone you need a central storage for configuration settings. Integrators, site administrators, or users should be able to configure your add-on product, or to add information that your add-on product requires to work (e.g. registration key for a third party service).

This tutorial will explain how you can easily create a Plone control panel that can be used to store add-on specific configuration data. It is based on a real world add-on product, called collective.akismet. collective.akismet provides akismet spam protection for plone.app.discussion comments. It needs to store the site specific akismet key in order to use the third party akismet service.


plone.registry vs. plone.app.registry
-------------------------------------

plone.registry provides debconf-like (or about:config-like) settings registries for Zope applications. A registry, with a dict-like API, is used to get and set values stored in records. Each record contains the actual value, as well as a field that describes the record in more detail. At a minimum, the field contains information about the type of value allowed, as well as a short title describing the record's purpose.

plone.app.registry provides the Plone UI integration for plone.registry. plone.app.registry will allow us to create the control panel forms from a simple schema definition.
Prerequisits

Before you can start with the development of your control panel, you have to make sure that you use the correct KGS (known good set) of Python packages in your buildout. For the current Plone 4.0.2 you have to add this line to your buildout configuration::

  [buildout]
  extends =
      http://good-py.appspot.com/release/plone.app.registry/1.0b2?plone=4.0.2


Creating the package
--------------------

First we create a simple plone skeleton package with paster (make sure you have paster and ZopeSkel installed)::

  $ paster create -t plone collective.akismet


Test first
----------

If you do test driven development, you might want to go to the testing section of this tutorial first and start with writing tests before writing code.
Add plone.app.registry to the package dependencies

You have to add plone.app.registry to the list of package dependencies of the newly created package. You do this by adding it to the install_requires section of the setup.py file.

There is no need to add plone.registry as well since plone.registry is a dependency of plone.app.registry.

setup.py::

  install_requires=[
      ...
      'plone.app.registry',
  ]


Defining the schema
-------------------

plone.app.registry allows you to create all control panel forms simply by defining a zope schema interface. For the akismet service we will need two fields, "akismet_key" and "akismet_key_site". Both are simple text fields with a description and a default value. See the zope.schema documentation for more options (text field, password field, etc.).

interfaces.py::

    from z3c.form import interfaces

    from zope import schema
    from zope.interface import Interface

    from zope.i18nmessageid import MessageFactory

    _ = MessageFactory('collective.akismet')


    class IAkismetSettings(Interface):
        """Global akismet settings. This describes records stored in the
        configuration registry and obtainable via plone.registry.
        """

        akismet_key = schema.TextLine(title=_(u"Akismet (Wordpress) Key"),
                                      description=_(u"help_akismet_key",
                                                    default=u"Enter in your Wordpress key here to "
                                                             "use Akismet to check for spam in comments."),
                                      required=False,
                                      default=u'',)

        akismet_key_site = schema.TextLine(title=_(u"Site URL"),
                                      description=_(u"help_akismet_key_site",
                                                    default=u"Enter the URL to this site as per your "
                                                             "Akismet settings."),
                                      required=False,
                                      default=u'',)

Creating the control panel
--------------------------

plone.app.registry allows us to create the control panel from the schema we just defined. We create an edit form that uses the schema fields to automatically create the control panel user interface and a wrapper that wraps the form into a view.

controlpanel.py::

    from plone.app.registry.browser import controlpanel

    from collective.akismet.interfaces import IAkismetSettings, _


    class AkismetSettingsEditForm(controlpanel.RegistryEditForm):

        schema = IAkismetSettings
        label = _(u"Akismet settings")
        description = _(u"""""")

        def updateFields(self):
            super(AkismetSettingsEditForm, self).updateFields()


        def updateWidgets(self):
            super(AkismetSettingsEditForm, self).updateWidgets()

    class AkismetSettingsControlPanel(controlpanel.ControlPanelFormWrapper):
        form = AkismetSettingsEditForm

Register the control panel view

Now we have to register the control panel we just created.

configure.zcml::

    <include package="plone.app.registry" />

    <!-- Control panel -->
    <browser:page
        name="akismet-settings"
        for="Products.CMFPlone.interfaces.IPloneSiteRoot"
        class=".controlpanel.AkismetSettingsControlPanel"
        permission="cmf.ManagePortal"
        />

You can test this view if you type this into the URL field of your browser:

http://localhost:8080/@@akismet-settings

Generic Setup
-------------

Since we finished creating the control panel, we have to integrate it into Plone by writing a generic setup profile for the product.

We also add plone.app.registry as a dependency, to make sure it is automatically installed when we install our product.

configure.zcml::

    <configure
        xmlns="http://namespaces.zope.org/zope"
        xmlns:five="http://namespaces.zope.org/five"
        xmlns:genericsetup="http://namespaces.zope.org/genericsetup"
        xmlns:i18n="http://namespaces.zope.org/i18n"
        i18n_domain="collective.akismet">

      <include package="plone.app.registry" />

      <genericsetup:registerProfile
          name="default"
          title="Akismet spam protection"
          directory="profiles/default"
          description="Provides Akismet spam protection for plone.app.discussion comments."
          provides="Products.GenericSetup.interfaces.EXTENSION"
          />

    </configure>

Create a "profiles" and a "default" folder inside. Afterwards, we create a controlpanel.xml file where we register the akismet control panel so it shows up in the Plone control panel.

profiles/default/controlpanel.xml::

    <?xml version="1.0"?>
    <object
        name="portal_controlpanel"
        xmlns:i18n="http://xml.zope.org/namespaces/i18n"
        i18n:domain="collective.akismet"
        purge="False">

        <configlet
            title="Akismet"
            action_id="akismet"
            appId="collective.akismet"
            category="Products"
            condition_expr=""
            url_expr="string:${portal_url}/@@akismet-settings"
            visible="True"
            i18n:attributes="title">
                <permission>Manage portal</permission>
        </configlet>

    </object>

profiles/default/metadata.xml::

    <metadata>
     <version>1</version>
     <dependencies>
      <dependency>profile-plone.app.registry:default</dependency>
     </dependencies>
    </metadata>

Next, we tell plone.app.registry about the IAkismetSettings interface, so we can look it up easily.

profiles/default/registry.xml::

    <?xml version="1.0"?>
    <registry>
     <records interface="collective.akismet.interfaces.IAkismetSettings" />
    </registry>

Using the registry in Python code
---------------------------------

Now that we have set up the registry, we can use it in our application. We can retrieve the settings of the akismet registry by querying the registry for the IAkismetSettings interface:
Accessing the registry

To get or set the value of a record, you must first look up the registry itself. The registry is registered as a local utility, so we can look it up with::

    >>> from zope.component import getUtility
    >>> from plone.registry.interfaces import IRegistry

    >>> registry = getUtility(IRegistry)

Now we fetch the AkismetSetting registry

    >>> from collective.akismet.interfaces import IAkismetSettings
    >>> settings = registry.forInterface(IAkismetSettings)

And now we can access the values

    >>> self.settings.akismet_key
    >>> ''
    >>> self.settings.akismet_key_site
    >>> ''


Testing
-------

We won't go into any details about how to set up or run tests here. If you are unfamiliar with testing, please see the Plone testing tutorial. For the complete testing code, see https://svn.plone.org/svn/collective/collective.akismet/trunk/collective/akismet/tests/.

We create a RegistryTest class that inherits from PloneTestCase and uses the AkismetLayer we set up before. We login as portal owner and set up the AkismetSettings::


    class RegistryTest(PloneTestCase):

        layer = AkismetLayer

        def afterSetUp(self):
            # Set up the akismet settings registry
            self.loginAsPortalOwner()
            self.registry = Registry()
            self.registry.registerInterface(IAkismetSettings)

    First, we test if the akismet control panel view we created is accessable.

        def test_akismet_controlpanel_view(self):
            view = getMultiAdapter((self.portal, self.portal.REQUEST),
                                   name="akismet-settings")
            view = view.__of__(self.portal)
            self.failUnless(view())

Test that the akismet control panel view is protected and anonymous users can't view or edit the akismet control panel::

    def test_akismet_controlpanel_view_protected(self):
        from AccessControl import Unauthorized
        self.logout()
        self.assertRaises(Unauthorized,
                          self.portal.restrictedTraverse,
                         '@@akismet-settings')

Test if the two records we created for the akismet control panel can be retrieved from the AkismetSettings registry::

    def test_record_akismet_key(self):
        # Test that the akismet_key record is in the control panel
        record_akismet_key = self.registry.records[
            'collective.akismet.interfaces.IAkismetSettings.akismet_key']
        self.failUnless('akismet_key' in IAkismetSettings)
        self.assertEquals(record_akismet_key.value, u"")

    def test_record_akismet_key_site(self):
        record_akismet_key_site = self.registry.records[
            'collective.akismet.interfaces.IAkismetSettings.akismet_key_site']
        self.failUnless('akismet_key_site' in IAkismetSettings)
        self.assertEquals(record_akismet_key_site.value, u"")

As last step, set up the test loader::

    def test_suite():
        return unittest.defaultTestLoader.loadTestsFromName(__name__)

Further reading
---------------

You learned how to set up a add-on product registry from a zope interface definition and how to retrieve these settings in your Python application code.

See the plone.app.registry and plone.registry documentation for further information:

    http://pypi.python.org/pypi/plone.app.registry
    http://pypi.python.org/pypi/plone.registry

Example Usage

    https://github.com/collective/collective.akismet
    https://github.com/plone/plone.formwidget.recaptcha/
    https://github.com/plone/plone.app.discussion

Advanced

    Getting registry settings in Plone to display in fieldsets

