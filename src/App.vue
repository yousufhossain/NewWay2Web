<template>
  <v-app>
    <v-navigation-drawer fixed :mini-variant="miniVariant" :clipped="clipped"
      v-model="drawer" app>
      <v-list>
        <v-list-tile value="true" v-for="(item, i) in items" :key="i">
          <v-list-tile-action>
            <v-icon v-html="item.icon"></v-icon>
          </v-list-tile-action>
          <v-list-tile-content>
            <v-list-tile-title v-text="item.title"></v-list-tile-title>
          </v-list-tile-content>
        </v-list-tile>
      </v-list>
    </v-navigation-drawer>
    <v-toolbar fixed app :clipped-left="clipped">
      <v-toolbar-side-icon @click.stop="drawer = !drawer"></v-toolbar-side-icon>
      <v-btn icon @click.stop="miniVariant = !miniVariant">
        <v-icon v-html="miniVariant ? 'chevron_right' : 'chevron_left'"></v-icon>
      </v-btn>
      <v-btn icon @click.stop="clipped = !clipped">
        <v-icon>web</v-icon>
      </v-btn>
      <v-btn icon @click.stop="fixed = !fixed">
        <v-icon>remove</v-icon>
      </v-btn>
      <v-toolbar-title v-text="title"></v-toolbar-title>
      <v-spacer></v-spacer>
      <v-btn icon @click.stop="rightDrawer = !rightDrawer">
        <v-icon>menu</v-icon>
      </v-btn>
    </v-toolbar>
    <v-content>
      <v-container fluid>
        <v-layout row>
          <v-flex xs6>
            <v-text-field name="input-7-1" label="Enter your HTMl here" v-model="HTMLinput"
              textarea></v-text-field>
          </v-flex>
          <v-flex xs6>
            <v-text-field name="input-7-1" label="Your javascript" v-model="ConvertedJS"
              textarea readonly></v-text-field>
          </v-flex>
        </v-layout>
        <v-btn @click.native="ConvertHTMLtoJS">Convert</v-btn>
        <v-layout row>
          <v-flex xs6>
            <v-text-field name="input-7-1" label="Console" v-model="ConsoleOutput"
              textarea></v-text-field>
          </v-flex>
          <v-flex xs6>
            <v-text-field name="input-7-1" label="DOM object" v-model="DOMobject"
              textarea readonly></v-text-field>
          </v-flex>
        </v-layout>
        <div ref='parentToAppendTo'></div>
      </v-container>
    </v-content>
    <v-navigation-drawer temporary :right="right" v-model="rightDrawer" fixed>
      <v-list>
        <v-list-tile @click.native="right = !right">
          <v-list-tile-action>
            <v-icon>compare_arrows</v-icon>
          </v-list-tile-action>
          <v-list-tile-title>Switch drawer (click me)</v-list-tile-title>
        </v-list-tile>
      </v-list>
    </v-navigation-drawer>
    <v-footer :fixed="fixed" app>
      <span>&copy; 2017</span>
    </v-footer>
  </v-app>
</template>

<script>
  var htmlParser = require("htmlparser")
  export default {
    data () {
      return {
        ConsoleOutput: null,
        DOMobject: null,
        ConvertedJS: null,
        clipped: false,
        HTMLinput: null,
        drawer: true,
        fixed: false,
        items: [
          { icon: 'bubble_chart', title: 'HTML to Javascript' }
        ],
        miniVariant: false,
        right: true,
        rightDrawer: false,
        title: 'A new way 2 web',
        parsedDOM: null,
        objectDefinition: [],
        attrDefinition:[],
        appendDOMs: [],
        depth: 0,
        number: 0
      }
    },
    mounted: function() {
      debugger
      this.$refs.parentToAppendTo.appendChild(this.renderHTML())
    },
    methods: {
      ConvertHTMLtoJS() {
        var self = this
        self.objectDefinition = []
        self.attrDefinition = []
        var handler = new htmlParser.DefaultHandler(function (error, dom) {
          if (error)
            self.ConsoleOutput = JSON.stringify(error)
          else
            self.DOMobject = JSON.stringify(dom)
            self.ParseHTML(dom)
            self.ConvertedJS = self.objectDefinition.join("\r\n") + "\r\n" 
            + self.attrDefinition.join("\r\n") + "\r\n" + self.appendDOMs.join("\r\n")
        })

        var parser = new htmlParser.Parser(handler)
        
        parser.parseComplete(this.HTMLinput)
      },
      ParseHTML(dom) {
        var self = this
        var elementName = ''
        for(var element of dom) {
          if(element.hasOwnProperty('type')) {
            elementName = element.name + (100000 + self.depth * 1000 + self.number)
            
            if(element.type == "tag") {
              self.number++
              self.objectDefinition.push("var " + elementName + " = document.createElement('" + element.name + "')")
              
              if(element.hasOwnProperty('attribs')) {
                for(var attr of Object.keys(element.attribs)) {
                  self.attrDefinition.push(elementName + ".setAttribute('" + attr + "', '" + element.attribs[attr] + "')")
                }
              }

              if(element.hasOwnProperty("children")) {
                var seqNumber = 1
                for(var child of element.children) {
                  if(child.type == "tag") {
                    self.ParseChildren(child, seqNumber, self.depth + 1, elementName)
                    seqNumber++
                  } else if(child.type == "text") {
                    var cleanedText = child.raw.replace(/(\r\n|\n|\r)/gm,"").replace(/\s/g, '')
                    if(cleanedText != "") {
                      self.attrDefinition.push(elementName + ".setAttribute('innerHTML', '" + cleanedText + "')")
                    }
                  }
                }
              }
            } else if(element.type == "text") {
              var cleanedText = element.raw.replace(/(\r\n|\n|\r)/gm,"").replace(/\s/g, '')
              if(cleanedText != "") {
                self.attrDefinition.push(elementName + ".setAttribute('innerHTML', '" + cleanedText + "')")
              }
            }
          }
        }
      },
      ParseChildren(element, seqNumber, childDepth, parentElementName) {
        var self = this
        var elementName = ''
        if(element.hasOwnProperty('type')) {
          elementName = element.name + (100000 + childDepth * 1000 + seqNumber)

          self.appendDOMs.push(parentElementName + ".appendChild(" + elementName + ")")

          if(element.type == "tag") {
            self.objectDefinition.push("var " + elementName + " = document.createElement('" + element.name + "')")
            
            if(element.hasOwnProperty('attribs')) {
              for(var attr of Object.keys(element.attribs)) {
                self.attrDefinition.push(elementName + ".setAttribute('" + attr + "', '" + element.attribs[attr] + "')")
              }
            }

            if(element.hasOwnProperty("children")) {
              var seqNumber = 1
              for(var child of element.children) {
                if(child.type == "tag") {
                  self.ParseChildren(child, seqNumber, childDepth + 1, elementName)
                  seqNumber++
                } else if(child.type == "text") {
                  var cleanedText = child.raw.replace(/(\r\n|\n|\r)/gm,"").replace(/\s/g, '')
                  if(cleanedText != "") {
                    self.attrDefinition.push(elementName + ".setAttribute('innerHTML', '" + cleanedText + "')")
                  }
                }
              }
            }
          } else if(element.type == "text") {
            var cleanedText = element.raw.replace(/(\r\n|\n|\r)/gm,"").replace(/\s/g, '')
            if(cleanedText != "") {
              self.attrDefinition.push(elementName + ".setAttribute('innerHTML', '" + cleanedText + "')")
            }
          }
        }
      },
      renderHTML() {
        var div100000 = document.createElement('div')
        var button101001 = document.createElement('button')
        var div101002 = document.createElement('div')
        var a102001 = document.createElement('a')
        var a102002 = document.createElement('a')
        var a102003 = document.createElement('a')
        div100000.setAttribute('class', 'dropdown')
        button101001.setAttribute('class', 'btn btn-secondary dropdown-toggle')
        button101001.setAttribute('type', 'button')
        button101001.setAttribute('id', 'dropdownMenuButton')
        button101001.setAttribute('data-toggle', 'dropdown')
        button101001.setAttribute('aria-haspopup', 'true')
        button101001.setAttribute('aria-expanded', 'false')
        button101001.setAttribute('innerHTML', 'Dropdownbutton')
        div101002.setAttribute('class', 'dropdown-menu')
        div101002.setAttribute('aria-labelledby', 'dropdownMenuButton')
        a102001.setAttribute('class', 'dropdown-item')
        a102001.setAttribute('href', '#')
        a102001.setAttribute('innerHTML', 'Action')
        a102002.setAttribute('class', 'dropdown-item')
        a102002.setAttribute('href', '#')
        a102002.setAttribute('innerHTML', 'Anotheraction')
        a102003.setAttribute('class', 'dropdown-item')
        a102003.setAttribute('href', '#')
        a102003.setAttribute('innerHTML', 'Somethingelsehere')
        div100000.appendChild(button101001)
        div100000.appendChild(div101002)
        div101002.appendChild(a102001)
        div101002.appendChild(a102002)
        div101002.appendChild(a102003)

        return div100000
      }
    }
  }
</script>
