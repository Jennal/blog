# 解决cocos2dx-lua绑定print不能打印指针的问题

cocos2d-x的lua绑定，把原生的print函数给改掉了，就为了加个`cocos2d: [LUA-print] `的前缀，实在让人无语。临时工为了方便，直接去掉了table、user data可以显示指针地址的功能，所以我只好去改源代码了。

## 有2种改法，先说复杂的

找到这个文件

```
cocos2d-x/cocos/scripting/lua-bindings/manual/CCLuaStack.cpp
```

修改这个函数

```
lua_print
```

直接替换整个函数的代码

<pre class="cpp">
int lua_print(lua_State * luastate)
{
    int nargs = lua_gettop(luastate);

    std::string t;
    for (int i=1; i <= nargs; i++)
    {
        const char * str = lua_tostring(luastate, i);
        if (str)
        {
            /* number, string */
            t += lua_tostring(luastate, i);
        }
        else if (lua_isnone(luastate, i))
            t += "none";
        else if (lua_isnil(luastate, i))
            t += "nil";
        else if (lua_isboolean(luastate, i))
        {
            if (lua_toboolean(luastate, i) != 0)
                t += "true";
            else
                t += "false";
        }else{
            /* table, userdata, lightuserdata, thread */
            t += lua_typename(luastate, lua_type(luastate, i));
            char pointer[16];
            memset(pointer, 0, 16);
            sprintf(pointer, "%p", lua_topointer(luastate, i));
            t += ": ";
            t += pointer;
        }
        
        if (i!=nargs)
            t += "\t";
    }
    CCLOG("[LUA-print] %s", t.c_str());

    return 0;
}
</pre>

# 简单的办法

还是这个文件

```
cocos2d-x/cocos/scripting/lua-bindings/manual/CCLuaStack.cpp
```

找到这段代码

<pre class="cpp">
    // Register our version of the global "print" function
    const luaL_reg global_functions [] = {
        {"print", lua_print},
        {NULL, NULL}
    };
</pre>

直接注释掉，这样就可以用lua原生的print函数了。如果还是想要那个前缀，很简单，只需要在lua文件里面写就可以了，具体就不用多说了。