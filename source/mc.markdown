
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
	<title>Merry Christmas</title>
<style type="text/css">
body, html {
  
  font-family: Helvetica, Arial, sans-serif;
  color: #fff;
  font-size: 11px;
  width: 100%;
  height: 100%;
  background: #c00;

  background: -moz-radial-gradient(center, ellipse cover, #c00 0%, #500 100%);
  background: -webkit-gradient(radial, center center, 0px, center center, 100%, color-stop(0%,#c00), color-stop(100%,#500));
  background: -webkit-radial-gradient(center, ellipse cover, #c00 0%,#500 100%);
  background: -o-radial-gradient(center, ellipse cover, #c00 0%,#500 100%);
  background: -ms-radial-gradient(center, ellipse cover, #c00 0%,#500 100%);
  background: radial-gradient(center, ellipse cover, #c00 0%,#500 100%);
}

@-webkit-keyframes snow {
    0% { background-position: 0px 0px, 0px 0px, 0px 0px }
    /*50% { background-color: #b4cfe0 }*/
    100% {
        background-position: 500px 1000px, 400px 400px, 300px 300px;
        /*background-color: #6b92b9;*/
    }
}
@-moz-keyframes snow {
    0% { background-position: 0px 0px, 0px 0px, 0px 0px }
    /*50% { background-color: #b4cfe0 }*/
    100% {
        background-position: 500px 1000px, 400px 400px, 300px 300px;
        background-color: #6b92b9;
    }
}
@-ms-keyframes snow {
    0% { background-position: 0px 0px, 0px 0px, 0px 0px }
    /*50% { background-color: #b4cfe0 }*/
    100% {
        background-position: 500px 1000px, 400px 400px, 300px 300px;
        /*background-color: #6b92b9;*/
    }
}
@keyframes snow {
    0% { background-position: 0px 0px, 0px 0px, 0px 0px }
    /*50% { background-color: #b4cfe0 }*/
    100% {
        background-position: 500px 1000px, 400px 400px, 300px 300px;
        /*background-color: #6b92b9;*/
    }
}


body {
  -webkit-perspective: 5000px;
     -moz-perspective: 5000px;
      -ms-perspective: 5000px;
       -o-perspective: 5000px;
          perspective: 5000px;

  -webkit-perspective-origin: 0 20%;
     -moz-perspective-origin: 0 20%;
      -ms-perspective-origin: 0 20%;
       -o-perspective-origin: 0 20%;
          perspective-origin: 0 20%;

  -webkit-animation: snow 20s linear infinite;
     -moz-animation: snow 20s linear infinite;
      -ms-animation: snow 20s linear infinite;
          animation: snow 20s linear infinite;

  background-image: url('snow1.png'), url('snow2.png'), url('snow3.png');
}

@-webkit-keyframes spin {
  0% { -webkit-transform: rotateY( 0deg ); }
  100% { -webkit-transform: rotateY( 360deg ); }
}

@-moz-keyframes spin {
	0% { -moz-transform: rotateY( 0deg ); }
	100% { -moz-transform: rotateY( 360deg ); }
}

@-ms-keyframes spin {
  0% { -ms-transform: rotateY( 0deg ); }
  100% { -ms-transform: rotateY( 360deg ); }
}

@-o-keyframes spin {
  0% { -o-transform: rotateY( 0deg ); }
  100% { -o-transform: rotateY( 360deg ); }
}

@keyframes spin {
  0% { transform: rotateY( 0deg ); }
  100% { transform: rotateY( 360deg ); }
}

#tree {
  margin: 0 auto;
  position: relative;

  -webkit-animation: spin 20s infinite linear;
  -moz-animation: spin 20s infinite linear;
  -ms-animation: spin 20s infinite linear;
  -o-animation: spin 20s infinite linear;
  animation: spin 20s infinite linear;

  -webkit-transform-origin: 50% 0;
  -moz-transform-origin: 50% 0;
  -ms-transform-origin: 50% 0;
  -o-transform-origin: 50% 0;
  transform-origin: 50% 0;

  -webkit-transform-style: preserve-3d;
  -moz-transform-style: preserve-3d;
  -ms-transform-style: preserve-3d;
  -o-transform-style: preserve-3d;
  transform-style: preserve-3d;
}

#tree * {
  position: absolute;

  -webkit-transform-origin: 0 0;
     -moz-transform-origin: 0 0;
      -ms-transform-origin: 0 0;
       -o-transform-origin: 0 0;
          transform-origin: 0 0;
}

#branch {
  text-overflow: ellipsis;
  white-space: nowrap;
  overflow: hidden;
}

#text {
  padding-top: 6px;
  font-weight: bold;
}


.star {
  border-color: #f1c40f transparent transparent transparent;
  border-style: solid;
  border-top-width: 35.21127px;
  border-right-width: 50px;
  border-left-width: 50px;
  height: 0;
  margin-top: 35.21127px;
  margin-bottom: 22.63581px;
  position: relative;
  width: 0;
}
.star:before, .star:after {
  border-color: #f1c40f transparent transparent transparent;
  border-style: solid;
  border-top-width: 35.21127px;
  border-right-width: 50px;
  border-left-width: 50px;
  content: '';
  display: block;
  height: 0;
  left: -50px;
  position: absolute;
  top: -35.21127px;
  width: 0;
}
.star:before {
  -webkit-transform: rotate(70deg);
          transform: rotate(70deg);
}
.star:after {
  -webkit-transform: rotate(-70deg);
          transform: rotate(-70deg);
}

.star {
  left: 200px;
  top: 10px;
}


.title {
  font-size: 72px;
  font-weight: bold;
  width: 100%;
  text-align: center;
  top: 150px;
  word-spacing: 50px;
  position: relative;
}

.poem {
  font-size: 24px;
}

.poem,
.footer {
  position: relative;
  top: 200px;
  width: 100%;
  text-align: center;
}

.footer{
	border-top: 1px solid #eee;
	margin:50px 0px;
}

</style>
</head>
<body>
<audio src="https://kidsongs.com/media/downloadable/files/samples/f/i/file_5_96.mp3" autoplay></audio>

<div id="tree">
	<div class="star"></div>
</div>

<div class="poem">
 <p>I wish you a Merry Christmas 2 @Cloris
</div>
<div class="footer">
 
 <p> @2016 <a href="http://coolshell.cn/christmas/">  CH</a>
</div>

</body>
</html>

<script type="text/javascript">

	var greeting = [
		'{0}',
	];

	var str = ["Just 4 Cloris!", "Merry Christmas!", "Happy New Year!", "Hello 2017!"];

	if (!String.prototype.format) {
	  String.prototype.format = function() {
	    var args = arguments;
	    return this.replace(/{(\d+)}/g, function(match, number) { 
	      return typeof args[number] != 'undefined'
	        ? args[number]
	        : match
	      ;
	    });
	  };
	}

	function transform( element, value ) {
		element.style.WebkitTransform = value;
		element.style.MozTransform = value;
		element.style.msTransform = value;
		element.style.OTransform = value;
		element.style.transform = value;
	}

	function getRandom(min, max) {
	    return Math.floor(Math.random() * (max - min + 1) + min);
	}


	function createBranch(width, height) {
		div = document.createElement( 'div' );
		span = document.createElement( 'span' );

		s = greeting[getRandom(0, greeting.length - 1)].format(str[getRandom(0, str.length - 1)]);
		
		text = document.createTextNode(s); 

		div.setAttribute("id", "branch");
		span.setAttribute("id", "text");
		
  		span.appendChild(text);
  		div.appendChild(span);

		div.style.width = width + 'px';
		div.style.height = height + 'px';
		var green = 50+Math.ceil(Math.random() * 200);
		var other = Math.ceil(Math.random() * 50);
		//console.log("rgba("+other+","+green+","+other+", 1)");
		div.style.background = "rgba("+other+","+green+","+other+", 1)";
		//div.style.position = "relative";
		return div;
	}

		
	var width = 500;
	var height = 600;
	var tree = document.getElementById("tree");
	tree.style.width = width + 'px';
	tree.style.height = height + 'px';
	//tree.style.margin = "auto";
	//tree.style.background = "#fefefe";

	for ( i = 0; i<300; i++) {
		var top_margin = 70;
		var x = width/2;
		var y = Math.round( Math.random() * height ) + top_margin;
		var rx = 0;
		var ry = Math.random() * 360;
		var rz = 0;//-Math.random() * 15;
		var elementWidth = 15 + ( ( (y - top_margin ) / height ) * width / 1.8 );
		var elementHeight = 26;

		//console.log(x, y, rx, ry, rz, elementWidth,  elementHeight)
		var div =  createBranch(elementWidth, elementHeight);

		transform(div, 'translate3d('+x+'px, '+y+'px, 0px) rotateX('+rx+'deg) rotateY('+ry+'deg) rotateZ('+rz+'deg)');
	  	tree.appendChild( div ); 
  	}

</script>

