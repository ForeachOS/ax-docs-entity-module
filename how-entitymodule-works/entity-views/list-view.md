[[list-view]]
=== List view

default settings of a list view:

- user must have read allowable action
- renders values as LIST_VALUE
- shows all readable properties
- adds update/delete buttons for every item if the user as update and delete action respectively
- supports paging and sorting
- allows configuring sortable properties and the default sort
- includes a form at the top that can be used for adding filters
- a create button for the entity if the user has create action allowed
- supports global feedback messages set with the `EntityViewPageHelper`

default processors

==== Adding a list view
TODO

==== Customizing a list view
TODO

.Custom sorting
Customizing the sort behaviour of a specific property can be done by setting a `Sort.Order.class` attribute on the `EntityPropertyDescriptor`.
You can use this to sort on a different property instead, or to specify behaviour for null handling and case sensitivity.

[[list-view-filter]]
==== Filtering a list view

By default a list view is not filtered.
If your entity has an <<entity-query-executor,`EntityQueryExecutor`>> available you can easily activate a simple yet powerful <<entity-query-language,EQL-based filter>>.
Activating `EntityQuery` based filtering can be done through the builders:

[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
// Activate the EQL-based filter on every entity that has an EntityQueryExecutor
configuration.matching( c -> c.hasAttribute( EntityQueryExecutor.class ) )
             .listView( lvb -> lvb.entityQueryFilter( true ) );
----

See <<entity-list-view-custom-filter,this chapter>> for information on how to configure a custom filter for a list view.


[[entity-list-view-custom-filter]]
==== Adding a custom filter to a list view

A filter on a list view usually consists of 2 parts:

* an `EntityViewProcessor` that provides the filtering options on the list view
* an `EntityViewProcessor` that fetches the items using the filtering options
** a custom form is usually added to the `EntityViewCommand.addExtension()` for both postback and optional validation

Both parts can easily be combined in a single `EntityViewProcessor`.

*Custom filter example*

The following code illustrates adding a simple filter to a view.
The filter uses a separate repository method to lookup entities by name.
The filter options are added as a form on top of the list view, the form in this case rendered via a custom Thymeleaf template.

.Implementation of a single class that holds all filter logic
[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
private static class GroupFilteringProcessor extends EntityViewProcessorAdapter
{
	@Autowired
	private GroupRepository groupRepository;

	@Override
    public void initializeCommandObject( EntityViewRequest entityViewRequest,
                                         EntityViewCommand command,
                                         WebDataBinder dataBinder ) {
        command.addExtension( "filter", "" );
    }

    @Override
    protected void doControl( EntityViewRequest entityViewRequest,
                              EntityView entityView,
                              EntityViewCommand command,
                              BindingResult bindingResult,
                              HttpMethod httpMethod ) {
        String filter = command.getExtension( "filter", String.class );
        Pageable pageable = command.getExtension( PageableExtensionViewProcessor.DEFAULT_EXTENSION_NAME, Pageable.class );

        if ( !StringUtils.isBlank( filter ) ) {
            entityView.addAttribute( "items", groupRepository.findByNameContaining( filter, pageable ) );
        }
        else {
            entityView.addAttribute( "items", groupRepository.findAll( pageable ) );
        }
    }

    @Override
    protected void postRender( EntityViewRequest entityViewRequest,
                               EntityView entityView,
                               ContainerViewElement container,
                               ViewElementBuilderContext builderContext ) {
        Optional<ContainerViewElement> header = find( container, "entityListForm-header", ContainerViewElement.class );
        header.ifPresent(
                h -> {
                    Optional<NodeViewElement> actions
                            = find( h, "entityListForm-header-actions", NodeViewElement.class );
                    actions.ifPresent( a -> a.addCssClass( "pull-right" ) );

                    h.addChild( new TemplateViewElement( "th/entityModuleTest/filters :: filterForm" ) );
                }
        );
    }
}
----

.Custom Thymeleaf template that builds the form
[source,xml,indent=0]
[subs="verbatim,quotes,attributes"]
----
<fragments xmlns:th="http://www.w3.org/1999/xhtml">
    <div class="list-header form form-inline" th:fragment="filterForm">
        <div class="form-group">
            <label for="group-name-filter">Filter by name:</label>
            <input id="group-name-filter" name="extensions[filter]" th:value="${entityViewCommand.extensions['filter']}" type="text" class="form-control" />
        </div>
        <input type="submit" class="btn btn-default" value="Apply filter" />
    </div>
</fragments>
----

.Registration of the custom filter on the list view
[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
entities.withType( Group.class )
        .listView( lvb -> lvb
            .entityQueryFilter( false )           // optional - disable the previously activated entity query filter
            .filter( groupFilteringProcessor() )  // register the custom filter
		);
----

==== List summary view

It is possible to activate a detail view inline in a list view.
If the `EntityConfiguration` or `EntityAssociation` has a view named *listSummaryView* a summary pane will automatically become available when clicking on the item row in the table.
The summary pane is called using AJAX and only the _content_ fragment of the page will be rendered.

[source,java,indent=0]
[subs="verbatim,quotes,attributes"]
----
// Activate a summary view in the main user results table using a custom Thymeleaf template
configuration.withType( User.class )
             .view( EntityView.SUMMARY_VIEW_NAME, vb -> vb.template( "th/myModule/userSummary" ) );
----
