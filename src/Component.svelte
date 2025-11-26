<script>
  import { getContext, onMount } from "svelte";
  import ApexTree from "apextree";

  const { styleable } = getContext("sdk");
  const component = getContext("component");
  let container;

  export let dataProvider;

  const options = {
    height: 800,
    width: 1000,
    nodeWidth: 150,
    nodeHeight: 120,
    childrenSpacing: 80,
    siblingSpacing: 40,
    direction: "top",

    // Required: determines what object is passed into nodeTemplate
    contentKey: "data",

    nodeTemplate: (content) => {
      const name = content.name || "";
      const role = content.role || "";
      const dept = content.department || "";
      const img = content.imageURL || "";

      return `
        <div class="org-node" style="align-items: justify-content">

          ${img
            ? `<img class="org-node-img" src="${img}" style="width: 60px; height: 60px"/>`
            : ``}

          <div class="org-node-name">${name}</div>
          <div class="org-node-role">${role}</div>
          <div class="org-node-dept">${dept}</div>

        </div>
      `;
    },

    canvasStyle: "border: 1px solid #ccc; background: #f6f6f6;"
  };

  function buildOrgTree(rows) {
    // Map userId => rowId because reports_to uses user IDs
    const userToRow = {};
    rows.forEach(row => {
      const userId = row.users_table?.[0]?._id;
      if (userId) userToRow[userId] = row._id;
    });

    // Build core node objects in ApexTree format
    const nodes = {};

    rows.forEach(row => {
      const imageURL =
        typeof row.profile_picture === "string"
          ? row.profile_picture
          : row.profile_picture?.url || null;

      nodes[row._id] = {
        id: row._id,

        // This is the object ApexTree will pass into nodeTemplate
        data: {
          id: row._id,
          name: row.name || "",
          role: row.role || "",
          department: row.Department || "",
          email: row.users_table?.[0]?.primaryDisplay || "",
          imageURL
        },

        // Optional styling per-node (not required)
        options: {},

        children: []
      };
    });

    let root = null;

    // Link children to managers
    rows.forEach(row => {
      const node = nodes[row._id];
      const managerUserId = row.reports_to?.[0]?._id;

      if (!managerUserId) {
        root = node;
        return;
      }

      const managerRowId = userToRow[managerUserId];
      if (managerRowId && nodes[managerRowId]) {
        nodes[managerRowId].children.push(node);
      }
    });

    return root;
  }

  onMount(() => {
    const orgObject = buildOrgTree(dataProvider.rows);
    const tree = new ApexTree(container, options);
    tree.render(orgObject);
  });
</script>

<div use:styleable={$component.styles}>
  <div bind:this={container}></div>
</div>

<style>

</style>

<!-- TODO:
Find a grownup who can help me select columns to behave as Title, Subtitle, Image, Child. 
-->