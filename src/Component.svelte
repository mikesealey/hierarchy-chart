<script>
  import { getContext, onMount } from "svelte";
  import ApexTree from "apextree";

  const { styleable } = getContext("sdk");
  const component = getContext("component");
  let container;

  // Settings from schema
  export let records;
  export let dataProvider;
  export let text1Column;
  export let text2Column;
  export let text3Column;
  export let imageColumn;
  export let reportsToColumn;

  console.log(dataProvider)

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
      const text1 = content.text1 || "";
      const text2 = content.text2 || "";
      const text3 = content.text3 || "";
      const img = content.imageURL || "";

      return `
        <div class="org-node" style="align-items: justify-content">

          ${img
            ? `<img class="org-node-img" src="${img}" style="width: 60px; height: auto; "/>`
            : ``}

          <div class="org-node-name">${text1}</div>
          <div class="org-node-role">${text2}</div>
          <div class="org-node-dept">${text3}</div>

        </div>
      `;
    },

    canvasStyle: "border: 1px solid #ccc; background: #f6f6f6;"
  };

  function getCell(row, key) {
    if (!key) return undefined;
    return row?.[key];
  }

  function getFirstRelId(value) {
    // relationship values are typically arrays of related rows
    if (!value) return undefined;
    if (typeof value === "string") return value; // sometimes stored as id
    const rel = Array.isArray(value) ? value[0] : value;
    if (rel && typeof rel === "object") {
      return rel._id || rel.id;
    }
    return undefined;
  }

  function normaliseImage(value) {
    // Accept string URL, attachment object with url, or array
    if (!value) return null;
    if (typeof value === "string") return value;
    if (Array.isArray(value)) return normaliseImage(value[0]);
    if (typeof value === "object" && value.url) return value.url;
    return null;
  }

  function looksLikeRelationship(val) {
    if (!val) return false;
    if (Array.isArray(val)) {
      return val.length > 0 && typeof val[0] === "object" && ("_id" in val[0] || "id" in val[0]);
    }
    return typeof val === "object" && ("_id" in val || "id" in val);
  }

  function detectSelfRelationKey(rows, excludeKey) {
    const preferred = ["users_table", "user", "users", "owner", "person"];
    for (const key of preferred) {
      if (key === excludeKey) continue;
      if (rows.some(r => looksLikeRelationship(r?.[key]))) {
        return key;
      }
    }
    // Fallback: find any relationship-like key present on many rows
    const counts = {};
    rows.forEach(r => {
      Object.keys(r || {}).forEach(k => {
        if (k === excludeKey) return;
        if (looksLikeRelationship(r[k])) {
          counts[k] = (counts[k] || 0) + 1;
        }
      });
    });
    let bestKey = null;
    let bestCount = 0;
    for (const [k, count] of Object.entries(counts)) {
      if (count > bestCount) {
        bestKey = k;
        bestCount = count;
      }
    }
    return bestCount > 0 ? bestKey : null;
  }

  function buildOrgTree(rows) {
    if (!Array.isArray(rows) || rows.length === 0) {
      return null;
    }

    // Build node map
    const nodes = {};
    const externalIdToRowId = {};
    const selfRelKey = detectSelfRelationKey(rows, reportsToColumn);
    rows.forEach(row => {
      console.log("HERE", getCell(row, text2Column))
      console.log("HERE", getCell(row, text3Column))
      const text1 = getCell(row, text1Column) ?? "";
      const text2 = getCell(row, text2Column) ?? "";
      const text3 = getCell(row, text3Column) ?? "";
      const imageURL = normaliseImage(getCell(row, imageColumn));
      if (selfRelKey) {
        const selfRelVal = getCell(row, selfRelKey);
        const externalId = getFirstRelId(selfRelVal);
        if (externalId) {
          externalIdToRowId[externalId] = row._id;
        }
      }

      nodes[row._id] = {
        id: row._id,
        data: {
          id: row._id,
          text1,
          text2,
          text3,
          imageURL
        },
        options: {},
        children: []
      };
    });

    // Link children to parents via selected relationship column
    let root = null;
    rows.forEach(row => {
      const node = nodes[row._id];
      const relVal = getCell(row, reportsToColumn);
      const parentRefId = getFirstRelId(relVal);
      // Resolve parent: prefer direct same-table id, else map via externalIdToRowId
      let parentRowId = parentRefId;
      if (parentRowId && !nodes[parentRowId] && externalIdToRowId[parentRowId]) {
        parentRowId = externalIdToRowId[parentRowId];
      }

      if (parentRowId && nodes[parentRowId] && parentRowId !== row._id) {
        nodes[parentRowId].children.push(node);
      } else if (!parentRefId) {
        // No parent — treat as root (keep first one)
        if (!root) root = node;
      }
    });

    // If we never found an explicit root, pick any top-level node
    if (!root) {
      // Find a node that is not a child of any other
      const childIds = new Set();
      Object.values(nodes).forEach(n => n.children.forEach(c => childIds.add(c.id)));
      root = Object.values(nodes).find(n => !childIds.has(n.id)) || Object.values(nodes)[0];
    }

    return root;
  }

  let tree;

  onMount(() => {
    tree = new ApexTree(container, options);
    const orgObject = buildOrgTree(records?.rows || dataProvider?.rows || []);
    if (orgObject) {
      tree.render(orgObject);
    }
  });

  // Track dependencies to trigger re-render on any relevant change
  let _deps;
  $: _deps = [records, dataProvider, text1Column, text2Column, reportsToColumn];

  // Re-render when inputs change
  $: if (tree && _deps) {
    const orgObject = buildOrgTree(records?.rows || dataProvider?.rows || []);
    if (orgObject) {
      tree.render(orgObject);
    }
  }
</script>

<div use:styleable={$component.styles}>
  <div bind:this={container}></div>
</div>

<style>
.org-node-img {
  border: 3px solid red;
}
</style>


