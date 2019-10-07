# CPPND: Memory Management Chatbot

Initial project codes have been modified as required to meet project rubric. Below are the detailed descriptions about relevant code changes.

## Exclusive Ownership

### `_chatLogic` ownership

`_chatLogic` is now owned exclusively by `ChatbotPanelDialog`. This is achieved by using unique smart pointer `std::unique_ptr<ChatLogic> _chatLogic;`.

### `GraphNode` ownership

`GraphNode` instance in the vector `_nodes` are execlusivly owned by `ChatLogic`. This is realized by use of vector of unqiue smart pointer `std::vector<std::unique_ptr<GraphNode>> _nodes;`

### `GraphNode` ownership is not transferred

GraphNode instance is  exclusvisely owned by `ChatLogic` in the form of `unique_ptr`. When passing it in function like `std::find_if`, refence is used, something like below,

```
auto parentNode = std::find_if(_nodes.begin(), _nodes.end(), [&parentToken](std::unique_ptr<GraphNode> &node) { return node->GetID() == std::stoi(parentToken->second); });
auto childNode = std::find_if(_nodes.begin(), _nodes.end(), [&childToken](std::unique_ptr<GraphNode> &node) { return node->GetID() == std::stoi(childToken->second); });
```

### outgoing edges `_childEdges` ownership

GraphNodes exclusively own the outgoing GraphEdges and hold non-owning references to incoming GraphEdges.

This is mainly acheived by using both `unique_ptr` smart pointer and raw pointer. 

```
// data handles (owned)
std::vector<std::unique_ptr<GraphEdge>> _childEdges;  // edges to subsequent nodes

// data handles (not owned)
std::vector<GraphEdge *> _parentEdges; // edges to preceding nodes 
```


## The Rule Of Five



### Rule of Five implementation

Chatbot class's Rule of Five implementation is as below. 


```
ChatBot::ChatBot(const ChatBot& source){
	std::cout << "ChatBot Copy Constructor" << std::endl;
	_image = new wxBitmap(*source._image);
	_rootNode = source._rootNode;
	_chatLogic = source._chatLogic;
};

//copy assignment operator
ChatBot& ChatBot::operator=(const ChatBot& source){
	std::cout << "ChatBot Copy Assignment" << std::endl;
	if (this != &source){
		//delete old resources before copying
		if(_image != NULL){
			delete _image;
			_image = NULL;
		}
		_image = new wxBitmap(*source._image);
		_rootNode = source._rootNode;
		_chatLogic = source._chatLogic;
	}
	return *this;
}

//move constructor
//ChatBot::ChatBot(ChatBot&& other):ChatBot(){
ChatBot::ChatBot(ChatBot&& source){
	std::cout << "ChatBot Move Constructor" << std::endl;
	_image = source._image;
	_rootNode = source._rootNode;
	_chatLogic = source._chatLogic;
	source._image = NULL;
	source._rootNode = nullptr;
	source._chatLogic = nullptr;
	//update _chatLogic pointer to chatbot
	_chatLogic->SetChatbotHandle(this);

}

//move assignment operator
ChatBot& ChatBot::operator=(ChatBot&& source){
	std::cout << "ChatBot Move Assignment Operator" << std::endl;
	if (this != &source){
		//delete old resources before moving
		if(_image != NULL){
			//this step is for pedagogical purpose, as we know for sure, in this project,
			// _image variable is expected to be NULL at this point.
			delete _image;
			_image = NULL;
		}
		_image = source._image;
		_rootNode = source._rootNode;
		_chatLogic = source._chatLogic;
		source._image = NULL;
		source._rootNode = nullptr;
		source._chatLogic = nullptr;
		//update _chatLogic pointer to chatbot
		_chatLogic->SetChatbotHandle(this);
	}

	return *this;
}
```

### printing indicates Rule of five components

After each query is entered, console output shows which member is called, something like below:

```
ChatBot Constructor
ChatBot Move Constructor
ChatBot Move Assignment Operator
ChatBot Destructor
ChatBot Destructor 
```


## Moving Semantics


### move local `Chatbot` instance to root node

As we intend to pass `Chatbot` instance among current nodes, a local `Chatbot` instance is first created on stack and passed to rootnode.

```
// create instance of chatbot
ChatBot chatbot_temp("../images/chatbot.png");
// add pointer to chatlogic so that chatbot answers can be passed on to the GUI
chatbot_temp.SetChatLogicHandle(this);
// add chatbot to graph root node
chatbot_temp.SetRootNode(rootNode);
rootNode->MoveChatbotHere(std::move(chatbot_temp));
```

### move `_GraphEdge` instance to `GraphNode`
This is done by below codes,

```
// create new edge
std::unique_ptr<GraphEdge> edge(std::make_unique<GraphEdge>(id));
edge->SetChildNode((*childNode).get());
edge->SetParentNode((*parentNode).get());
//_edges.push_back(edge.get());

// find all keywords for current node
AddAllTokensToElement("KEYWORD", tokens, *edge);

// store reference in child node and parent node
(*childNode)->AddEdgeToParentNode(edge.get());
(*parentNode)->AddEdgeToChildNode(std::move(edge));
```




### `_ChatLogic` holds non-owning references to `Chatbot`

ChatLogic has no ownership relation to the ChatBot instance and only holds a non-owing reference to it. ChatLogic's non-owning pointer to ChatBot object is updated in the move semantic of chatbot object.


