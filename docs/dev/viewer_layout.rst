Native viewer layout
====================

Jdaviz implements viewer docking in ``components/viewer_layout.vue``.
The component owns tabs, splits, resizing, and maximization, and keeps viewer
widgets mounted in a flat layer keyed by viewer IDs. Moving a tab
changes its geometry, not its widget instance.

The Python application owns ``state.viewer_items`` (live viewer metadata) and
``state.viewer_layout`` (placement). Viewer labels, widget references, and close
policy belong to the metadata, not the layout tree.

Configuration format
--------------------

Existing YAML files and Python configuration dictionaries are unchanged.
``viewer_area`` remains a list of ``container`` objects with inline viewer
specifications. For example::

    viewer_area:
      - container: col
        children:
          - container: row
            viewers:
              - name: Image
                plot: imviz-image-viewer
                reference: image-viewer
              - name: Comparison
                plot: imviz-image-viewer
                reference: comparison-viewer

``row`` places viewers side by side, ``col`` places them vertically, and a leaf
``stack`` places them in tabs. Nested ``children`` and multiple top-level
containers keep their existing meaning. A stack with structural children falls
back to a column, as the previous renderer did. Empty containers, an empty list,
and an omitted ``viewer_area`` all work without a migration.

The application creates viewers in the original order, preserving generated IDs,
names, and references. As before, viewers on a container are created before its
children, while the children are displayed before the container's own viewers.
Multiple top-level containers are displayed in a row. The configuration itself
is not modified.

Internal layout
---------------

The application translates the configuration into ``state.viewer_layout``.
This internal document replaces ``stack_items`` and ``golden_layout_state``;
it is not a new configuration format. It contains ``version: 1`` and a ``root``
node, or ``root: null`` for an empty area. Nodes use:

* ``row`` or ``column`` with an ordered ``children`` list and relative child
  ``weight`` values, initially equal;
* ``stack`` with an ordered list of viewer IDs in ``viewers`` and an
  ``activeViewerId``.

An optional ``maximizedViewerId`` tracks the maximized stack's active tab.
Python validates frontend edits against the live viewers, normalizes weights,
and prunes empty nodes and single-child splits. Viewer creation and removal
update both the metadata and placement. In the browser, edit helpers work on
copies of the normalized layout; publishing normalizes the completed edit.
Root-edge docking also prunes its source before splitting the remaining root.

Renaming a reference normally preserves the viewer ID. The existing
``_update_viewer_reference_name(..., update_id=True)`` option also updates IDs
in the layout, including active and maximized tabs. Docking itself preserves
viewer IDs and widget instances.

Saved Golden Layout documents are obsolete. Saved layout import/export and
layout persistence across kernel restarts are not provided by this experiment.
Layout remains synchronized within the running kernel, including when the
frontend is remounted.

Controls and lifecycle
----------------------

Select a tab with a click or with Left/Right and Home/End. Drag a tab into another
header to reorder or join its tabs, into the center to join a stack, or to an edge
to split. Outer-layout edges place a viewer alongside the entire layout. An
overflow menu provides direct access to hidden tabs. Closing or moving the active
tab selects its previous neighbor, or its next neighbor if it was first.

Panes share a 50-pixel minimum size, including their header. Nested splits
reserve enough space for their panes; a smaller container scales these minimums
down without changing stored weights. There are no per-viewer size settings.

Drag a splitter or use its arrow keys to resize. Pointer movement previews the
change locally; release commits it and Escape cancels it. Dropping a tab outside
the layout cancels. Protected viewers have no close control. The maximize button
expands/restores the stack; dragging out of a maximized stack restores the layout.

Completed edits are validated in Python against live viewer membership. Viewer
close requests carry the post-close layout so geometry and the viewer registry
can change together. The browser close handler checks close policy and layout;
the existing ID-based removal API also permits removing protected viewers.
Both use the same removal operation. Validation and layout operations live in
``core/viewer_layout.py``.

Header changes update tab labels, close controls, and overflow without notifying
plots to resize. Resize notifications are sent when viewer placement, visibility,
or the displayed widget changes. Unrelated viewer metadata does not trigger
placement work when edited in place, and equivalent state snapshots do not
trigger plot resizing.

Delayed synchronization must not rewind newer browser edits.
The ``viewer_layout_reset`` signal forces rollback after a rejected browser edit,
including when the accepted layout has not changed and no layout trait update
would otherwise be sent.

Jdaviz's application, plugin, and viewer popout tools remain independent of
docking. Whole-stack dragging, whole-stack close, and floating dock windows are
not implemented. Outer application panels still use splitpanes.

Testing
-------

Python layout and application tests live in ``jdaviz/core/tests/test_viewer_layout*.py``.
The native component's unit and browser tests live in ``tests/frontend``; see its
README for commands. ``tests/integration`` contains a real-widget fixture and
browser smoke checks for Solara and JupyterLab, including resize/reveal, focus,
same-kernel remount, and JupyterLab popout. Its README describes setup and the
limits of the notebook content-visibility check.
