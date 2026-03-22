---

tags: ["gameproject","tool","unity"]
image: /images/Frogment_Thumbnail.png
title: 'Game Project: Frogment'

---
🎮 Link to game: [https://coahl.itch.io/frogment](https://coahl.itch.io/frogment)

My Contributions:

- **Node Path System**
- **Node Conntection Tool**
- **Player Pathfinding**


This was my second game project at The Game Assembly, also in 5 weeks in Unity (Version 2022.3). This was a puzzle game made for Android where you play as a frog character. You automatically move to a block by tapping on it, and there are puzzles to solve by pressing buttons or rotating blocks.

## Node connections 

![](/images/Frogment_Pathfind.gif "Pathfinding")

On this project my main task was working with the players pathfinding and nodepath system. 

Every walkable block has a WalkableNode script assigned to it. To not overcomplicate things, this script only contains references to connected nodes and what node type it is (regular, stairs or ladders). I also added gizmo lines to show the node connections, which made it easy to detect whenever a connection was missing or should be removed

```cs
public class WalkableNode : MonoBehaviour
{
    [SerializeField] NodeType myType;
    public NodeType Type => myType;

    public List<WalkableNode> myConnectedNodes = new();

    Vector3 myLastPosition;
    Vector3 myLastForward;

    [HideInInspector] public bool myIsBound;
    private void Awake()
    {
        myLastPosition = transform.position;
    }

    public void ConnectNode(WalkableNode aNode)
    {
        if(myConnectedNodes.Contains(aNode))
        {
            Debug.Log("Aldready has node " + aNode.name + " in list", this);
            return;
        }

        myConnectedNodes.Add(aNode);
    }
    public void DisconnectNode(WalkableNode aNode)
    {
        if (!myConnectedNodes.Contains(aNode))
        {
            Debug.Log("Does not have node " + aNode.name + " in it's list", this);
            return;
        }

        myConnectedNodes.Remove(aNode);
    }

    public bool IsConnectedWith(WalkableNode aOther)
    {
        bool meToOther = myConnectedNodes.Contains(aOther);
        bool otherToMe = aOther.myConnectedNodes.Contains(this);

        if(meToOther && !otherToMe)
        {
            Debug.LogWarning("This node has a 1 way connection with node " + aOther.name, this);
        }
        else if(!meToOther && otherToMe)
        {
            Debug.LogWarning("This node has a 1 way connection with node " + name, aOther);
        }

        return meToOther && otherToMe;
    }

    /// <summary>
    /// Returns the offseted position the player should walk to when this is the current node
    /// </summary>
    public Vector3 GetWalkPosition()
    {
        switch(myType)
        {
            default:
            case NodeType.Regular:
                return transform.position + transform.up;
            case NodeType.Ladder:
                return transform.position + 0.5f * transform.up + 0.75f * transform.forward;
            case NodeType.Stair:
                return transform.position + 0.5f * transform.up;
                
        }
    }

    private void OnDrawGizmos()
    {
        
        Gizmos.color = Color.black;
        Gizmos.DrawWireSphere(GetWalkPosition(), 0.15f);
        Gizmos.color = Color.green;

        if(myConnectedNodes == null)
        {
            return;
        }

        foreach (var node in myConnectedNodes)
        {
            switch(myType)
            {
                default:
                    Gizmos.DrawLine(GetWalkPosition() + Vector3.up * 0.15f, node.GetWalkPosition());
                    break;
                case NodeType.Ladder:
                    Gizmos.DrawLine(GetWalkPosition() + transform.forward * 0.15f, node.GetWalkPosition());
                    break;
                
            }
            
        }
    }
}

public enum NodeType
{
    Regular,
    Ladder,
    Stair,
}
```

## Connection Tool

![](/images/Frogment_Connection.gif "Connections")

Node connections was then configured in the editor. Since doing that manually for every block in a level would take ages, I made a EditorWindow with node connections functions. I started of with a button that would remove all nodes and a button that would automatically connect nodes that should connect (depending on their placement & node type). Later I added a button to auto disconnect nodes, a button to check if node has snapped position and a button that would remove invalid connections

```cs
public class NodeWindow : EditorWindow
{
    [MenuItem("Tools/NodeWindow")]
    public static void CreateWindow()
    {
        NodeWindow window = GetWindow<NodeWindow>();
        window.titleContent = new GUIContent("NodeWindow");
        window.minSize = new Vector2(200,500);
    }

    private void OnGUI()
    {
        GUILayout.BeginVertical();
        GUILayout.FlexibleSpace();
        GUILayout.EndVertical();

        GUILayout.BeginHorizontal();
        GUILayout.FlexibleSpace();

        GUIContent buttonContent = new GUIContent("Clear all connections", "Removes all node connections!");
        if (GUILayout.Button(buttonContent, GUILayout.Width(130), GUILayout.Height(50)))
        {
            WalkableNode[] allNodes = FindObjectsOfType<WalkableNode>();
            foreach (WalkableNode node in allNodes)
            {
                EditorUtility.SetDirty(node);
                if (node.myConnectedNodes == null)
                {
                    node.myConnectedNodes = new();
                    continue;
                }

                node.myConnectedNodes.Clear();
            }
        }
        GUILayout.FlexibleSpace();
        GUILayout.EndHorizontal();

        GUILayout.BeginVertical();
        GUILayout.FlexibleSpace();
        GUILayout.EndVertical();

        GUILayout.BeginHorizontal();
        GUILayout.FlexibleSpace();

        buttonContent = new GUIContent("Auto connect", "Goes through all Walkable Nodes and conncects it with neighbouring nodes it should be connected with");
        if (GUILayout.Button(buttonContent, GUILayout.Width(100), GUILayout.Height(50)))
        {
            WalkableNode[] allNodes = FindObjectsOfType<WalkableNode>();
            foreach (WalkableNode node in allNodes)
            {
                EditorUtility.SetDirty(node);
                NodeUtilities.AutoConnectNode(node);
            }
        }


        GUILayout.FlexibleSpace(); // Add flexible space to the right of the button
        GUILayout.EndHorizontal();

        GUILayout.BeginVertical();
        GUILayout.FlexibleSpace();
        GUILayout.EndVertical();

        GUILayout.BeginHorizontal();
        GUILayout.FlexibleSpace();

        buttonContent = new GUIContent("Auto disconnect", "Goes through all Walkable Nodes and disconncects it with neighbouring nodes it should be connected with");
        if (GUILayout.Button(buttonContent, GUILayout.Width(120), GUILayout.Height(50)))
        {
            WalkableNode[] allNodes = FindObjectsOfType<WalkableNode>();
            foreach (WalkableNode node in allNodes)
            {
                EditorUtility.SetDirty(node);
                NodeUtilities.AutoDisconnectNode(node);
            }
        }

        GUILayout.FlexibleSpace(); // Add flexible space to the right of the button
        GUILayout.EndHorizontal();

        GUILayout.BeginVertical();
        GUILayout.FlexibleSpace();
        GUILayout.EndVertical();

        GUILayout.BeginHorizontal();
        GUILayout.FlexibleSpace();

        buttonContent = new GUIContent("Check every node", "Goes through all Walkable Nodes and detects any that are not snapped to a position");
        if (GUILayout.Button(buttonContent, GUILayout.Width(150), GUILayout.Height(50)))
        {
            WalkableNode[] allNodes = FindObjectsOfType<WalkableNode>();
            foreach (WalkableNode node in allNodes)
            {
                NodeUtilities.CheckNode(node);
            }
        }

        GUILayout.FlexibleSpace(); // Add flexible space to the right of the button
        GUILayout.EndHorizontal();

        GUILayout.BeginVertical();
        GUILayout.FlexibleSpace();
        GUILayout.EndVertical();

        GUILayout.BeginHorizontal();
        GUILayout.FlexibleSpace();

        buttonContent = new GUIContent("Fix every node", "Goes through all Walkable Nodes and removes connections to itself, null nodes and duplicate connections");
        if (GUILayout.Button(buttonContent, GUILayout.Width(150), GUILayout.Height(50)))
        {
            WalkableNode[] allNodes = FindObjectsOfType<WalkableNode>();
            foreach (WalkableNode node in allNodes)
            {
                EditorUtility.SetDirty(node);
                NodeUtilities.FixNode(node);
            }
        }

        GUILayout.FlexibleSpace(); // Add flexible space to the right of the button
        GUILayout.EndHorizontal();

        GUILayout.BeginVertical();
        GUILayout.FlexibleSpace();
        GUILayout.EndVertical();
    }
}

```