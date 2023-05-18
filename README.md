<!---
Dip98/Dip98 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
<html>
    <head>
        <meta charset="utf-8">
        <title>WIP [Not really]</title>
    </head>
    <body>

<canvas id='gl-canvas' width='600' height='600'></canvas>
<canvas id='ui-canvas' width='600' height='600'></canvas>

<style>
    
    body{
        overflow:hidden;
        margin:0;
        color:white;
    }
    
    #gl-canvas{
        
        display:none;
        
	}
    
</style>

<script id='geometry_vsh' type='GLSL3D'>#version 300 es

    precision lowp float;
    
    in vec3 vertPosition;
    in vec3 vertColor;
    
    out vec3 pixColor;
    
    uniform mat4 modelView;
    
    void main(){
        
        vec4 pos=modelView*vec4(vertPosition,1.0);
        pixColor=vertColor;
        gl_Position=pos;
    }
    
</script>

<script id='geometry_fsh' type='GLSL3D'>#version 300 es
    
    precision lowp float;
    
    in vec3 pixColor;
    out vec4 fragColor;
    
    void main(){
        
        fragColor=vec4(pixColor,1);
    }
    
</script>


<script src='https://cdnjs.cloudflare.com/ajax/libs/cannon.js/0.6.2/cannon.min.js'></script>

<script src="https://cdnjs.cloudflare.com/ajax/libs/gl-matrix/2.8.1/gl-matrix-min.js"></script>

<!--All of the div tags for the text-->
<div id = "time" style = "color:white; position:absolute; top:5px; left:5px;"></div>
<div id = "how-to" style = "position:absolute; background-color:rgb(194, 194, 194); color:black; padding:10px; width:200px; height:160px; text-align:center;"></div>
<div id = "heading" style = "position:absolute; color:white; padding:10px; text-align:center; font-size:25px"></div>
<div id = "version" style = "color:white; position:absolute; top:96%; left:5px;"></div>

<script type='application/javascript'>
    
function main(){
    let speed = 100;
    let timeSpeed = 0;
    let started = false;
    
    let world = new CANNON.World()
    
    world.broadphase = new CANNON.SAPBroadphase(world)
    world.broadphase.useBoundingBoxes = true
    
    //gravity
    world.gravity.set(0,-20,0)
    
    world.quatNormalizeSkip = 10
    world.quatNormalizeFast = true
    
    let solver = new CANNON.GSSolver()
    
    solver.iterations = 2
    solver.tolerance = 10000
    
    world.solver = solver
    
    world.defaultContactMaterial.friction = -10
    
    //bouciness of all surfaces numbers 0-1
    world.defaultContactMaterial.restitution=0.01
    
    var timeTextEl = document.getElementById("time");
    var how_toEl = document.getElementById("how-to");
    var headEl = document.getElementById("heading");
    var versionEl = document.getElementById("version");

    const glCanvas=document.getElementById('gl-canvas')
    const gl=glCanvas.getContext('webgl2',{preserveDrawingBuffer:true})
    
    const uiCanvas=document.getElementById('ui-canvas')
    const ctx=uiCanvas.getContext('2d')
    
    if(!gl){
        
        alert("You do not have WebGL, and this game requires it.")
        return
    }
    
    const width=window.innerWidth
    const height=window.innerHeight
    
    gl.viewport(0,0,width,height)
    
    const m4={
        
        identity:function(){
            
            return [1,0,0,0,0,1,0,0,0,0,1,0,0,0,0,1]
        },
        
        perspective:function(fov,aspect,zn,zf){
            
            var f=Math.tan(Math.PI*0.5-fov*0.5)
            
            var rangeInv=1.0/(zn-zf)
            return [f/aspect,0,0,0,0,f,0,0,0,0,(zn+zf)*rangeInv,-1,0,0,zn*zf*rangeInv*2,0]
        },
        mult:function(a,b){
            return [
                b[0]*a[0],
                b[1]*a[5],
                b[2]*a[10]+b[3]*a[14],
                -b[2],
                b[4]*a[0],
                b[5]*a[5],
                b[6]*a[10]+b[7]*a[14],
                -b[6],
                b[8]*a[0],
                b[9]*a[5],
                b[10]*a[10]+b[11]*a[14],
                -b[10],
                b[12]*a[0],
                b[13]*a[5],
                b[12]*a[2]+b[13]*a[6]+b[14]*a[10]+b[15]*a[14],
                -b[14]+b[15]*a[15],
            ]
        },
        
        translate:function(m,x,y,z){
            
            let a=m
            
            a[12]=x*a[0]+y*a[4]+z*a[8]+a[12]
            a[13]=x*a[1]+y*a[5]+z*a[9]+a[13]
            a[14]=x*a[2]+y*a[6]+z*a[10]+a[14]
            a[15]=x*a[3]+y*a[7]+z*a[11]+a[15]
            
            return a
        },
        
        xRotate:function(m,angle){
            
			let elems=m
			let c=Math.cos(-angle)
			let s=Math.sin(-angle)
			let t=elems[1]
			elems[1]=t*c+elems[2]*s
			elems[2]=t*-s+elems[2]*c
			t=elems[5]
			elems[5]=t*c+elems[6]*s
			elems[6]=t*-s+elems[6]*c
			t=elems[9]
			elems[9]=t*c+elems[10]*s
			elems[10]=t*-s+elems[10]*c
			t=elems[13]
			elems[13]=t*c+elems[14]*s
			elems[14]=t*-s+elems[14]*c
			return elems
        },
        
        yRotate:function(m,angle){
			var c=Math.cos(angle)
            var s=Math.sin(angle)
            return [c,0,-s,0,0,1,0,0,s,0,c,0,0,0,0,1]
        },
    }
    
    function random(a,b){
        
        return Math.random()*(b-a)+a
    }
    
    function constrain(x,a,b){
        
        return x<a?a:x>b?b:x
    }
    
    //Collapse this[
    const vertexShaderText=document.getElementById('geometry_vsh').text
    const fragmentShaderText=document.getElementById('geometry_fsh').text
    const vertexShader=gl.createShader(gl.VERTEX_SHADER)
    const fragmentShader=gl.createShader(gl.FRAGMENT_SHADER)
    gl.shaderSource(vertexShader,vertexShaderText)
    gl.shaderSource(fragmentShader,fragmentShaderText)
    gl.compileShader(vertexShader)
    gl.compileShader(fragmentShader)
    
    const program=gl.createProgram()
    gl.attachShader(program,vertexShader)
    gl.attachShader(program,fragmentShader)
    gl.linkProgram(program)
    gl.useProgram(program)
    
    gl.enable(gl.DEPTH_TEST)
    gl.depthFunc(gl.LEQUAL)
    gl.enable(gl.CULL_FACE)
    gl.cullFace(gl.BACK)
    
    let positionAttribLocation=gl.getAttribLocation(program,"vertPosition")
    let colorAttribLocation=gl.getAttribLocation(program,"vertColor")
    gl.enableVertexAttribArray(positionAttribLocation)
    gl.enableVertexAttribArray(colorAttribLocation)
    
    //light dir for all objects
    let lightDir=[1,3,1.5]
    
    let m=1/(lightDir[0]*lightDir[0]+lightDir[1]*lightDir[1]+lightDir[2]*lightDir[2])
    
    lightDir[0]*=m
    lightDir[1]*=m
    lightDir[2]*=m
    //]
    
    var cubeCollide = function(p1X, p1Y, p1Z, p1W, p1H, p1D, p2X, p2Y, p2Z, p2W, p2H, p2D){
    return(p1X + p1W / 2 > p2X - p2W && p1X - p1W / 2 < p2X + p2W && p1Y + p1H / 2 > p2Y - p2H && p1Y - p1H / 2 < p2Y + p2H && p1Z + p1D / 2 > p2Z - p2D && p1Z - p1D / 2 < p2Z + p2D); 
};
    
    class Mesh {
        
        constructor(verts,index){
            
            this.setMesh(verts||[],index||[])
        }
        
        setMesh(verts,index){
            
            this.mesh={
                data:{
                    verts:new Float32Array(verts),
                    index:new Uint16Array(index)
                },
                buffers:{
                    verts:gl.createBuffer(),
                    index:gl.createBuffer()
                },
                
                indexAmount:index.length,
            }
        }
        
        setMeshFromFunction(func){
            
            let verts=[],index=[],meshCollider=[]
            
            function addPlatform(x,y,z,w,h,l,rot,col){
                
                rot=rot||[0,0,0]
                
                let rotation=quat.fromEuler(quat.create(),rot[0],rot[1],rot[2])
                let model=mat4.create()
                
                mat4.fromRotationTranslation(model,rotation,[x,y,z,1])
                world.addBody(new CANNON.Body({
                    
                    shape:new CANNON.Box(new CANNON.Vec3(0.5*w,0.5*h,0.5*l)),mass:0,position:new CANNON.Vec3(x,y,z),quaternion:new CANNON.Quaternion(...rotation)
                }))
                
                let v=[
                    
                    vec4.fromValues(-0.5*w,0.5*h,-0.5*l,1),
                    vec4.fromValues(-0.5*w,0.5*h,0.5*l,1),
                    vec4.fromValues(0.5*w,0.5*h,0.5*l,1),
                    vec4.fromValues(0.5*w,0.5*h,-0.5*l,1),
                    vec4.fromValues(-0.5*w,-0.5*h,-0.5*l,1),
                    vec4.fromValues(-0.5*w,-0.5*h,0.5*l,1),
                    vec4.fromValues(0.5*w,-0.5*h,0.5*l,1),
                    vec4.fromValues(0.5*w,-0.5*h,-0.5*l,1),
                ]
                
                let shade=[]
                
                let normals=[
                    
                    vec4.fromValues(0,1,0,1),
                    vec4.fromValues(0,0,1,1),
                    vec4.fromValues(0,0,-1,1),
                    vec4.fromValues(1,0,0,1),
                    vec4.fromValues(-1,0,0,1),
                    vec4.fromValues(0,-1,0,1),
                ]
                
                for(var i=0,_l=v.length;i<_l;i++){
                    
                    vec4.transformMat4(v[i],v[i],model)
                    
                    if(i<6){
                        
                        vec4.transformQuat(normals[i],normals[i],rotation)
                        let n=normals[i]
                        let d=n[0]*lightDir[0]+n[1]*lightDir[1]+n[2]*lightDir[2]
                        shade[i]=d*0.8+0.65
                        
                    }
                }
                
                let vl=verts.length/6
                
                verts.push(
                    
                    v[0][0],v[0][1],v[0][2],col[0]*shade[0],col[1]*shade[0],col[2]*shade[0],
                    v[1][0],v[1][1],v[1][2],col[0]*shade[0],col[1]*shade[0],col[2]*shade[0],
                    v[2][0],v[2][1],v[2][2],col[0]*shade[0],col[1]*shade[0],col[2]*shade[0],
                    v[3][0],v[3][1],v[3][2],col[0]*shade[0],col[1]*shade[0],col[2]*shade[0],
                    
                    v[1][0],v[1][1],v[1][2],col[0]*shade[1],col[1]*shade[1],col[2]*shade[1],
                    v[2][0],v[2][1],v[2][2],col[0]*shade[1],col[1]*shade[1],col[2]*shade[1],
                    v[5][0],v[5][1],v[5][2],col[0]*shade[1],col[1]*shade[1],col[2]*shade[1],
                    v[6][0],v[6][1],v[6][2],col[0]*shade[1],col[1]*shade[1],col[2]*shade[1],
                    
                    v[0][0],v[0][1],v[0][2],col[0]*shade[2],col[1]*shade[2],col[2]*shade[2],
                    v[3][0],v[3][1],v[3][2],col[0]*shade[2],col[1]*shade[2],col[2]*shade[2],
                    v[4][0],v[4][1],v[4][2],col[0]*shade[2],col[1]*shade[2],col[2]*shade[2],
                    v[7][0],v[7][1],v[7][2],col[0]*shade[2],col[1]*shade[2],col[2]*shade[2],
                    
                    v[2][0],v[2][1],v[2][2],col[0]*shade[3],col[1]*shade[3],col[2]*shade[3],
                    v[3][0],v[3][1],v[3][2],col[0]*shade[3],col[1]*shade[3],col[2]*shade[3],
                    v[6][0],v[6][1],v[6][2],col[0]*shade[3],col[1]*shade[3],col[2]*shade[3],
                    v[7][0],v[7][1],v[7][2],col[0]*shade[3],col[1]*shade[3],col[2]*shade[3],
                    
                    v[0][0],v[0][1],v[0][2],col[0]*shade[4],col[1]*shade[4],col[2]*shade[4],
                    v[1][0],v[1][1],v[1][2],col[0]*shade[4],col[1]*shade[4],col[2]*shade[4],
                    v[4][0],v[4][1],v[4][2],col[0]*shade[4],col[1]*shade[4],col[2]*shade[4],
                    v[5][0],v[5][1],v[5][2],col[0]*shade[4],col[1]*shade[4],col[2]*shade[4],
                    
                    v[4][0],v[4][1],v[4][2],col[0]*shade[5],col[1]*shade[5],col[2]*shade[5],
                    v[5][0],v[5][1],v[5][2],col[0]*shade[5],col[1]*shade[5],col[2]*shade[5],
                    v[6][0],v[6][1],v[6][2],col[0]*shade[5],col[1]*shade[5],col[2]*shade[5],
                    v[7][0],v[7][1],v[7][2],col[0]*shade[5],col[1]*shade[5],col[2]*shade[5],
                    
                )
                
                index.push(
                    
                    0+vl,1+vl,2+vl,
                    0+vl,2+vl,3+vl,
                    5+vl,6+vl,7+vl,
                    6+vl,5+vl,4+vl,
                    8+vl,9+vl,10+vl,
                    11+vl,10+vl,9+vl,
                    14+vl,13+vl,12+vl,
                    13+vl,14+vl,15+vl,
                    18+vl,17+vl,16+vl,
                    17+vl,18+vl,19+vl,
                    22+vl,21+vl,20+vl,
                    23+vl,22+vl,20+vl
                )
            }
            
            
            func(addPlatform)
            
            this.setMesh(verts,index)
        }
        
        setBuffers(){
            
            gl.bindBuffer(gl.ARRAY_BUFFER,this.mesh.buffers.verts)
            gl.bufferData(gl.ARRAY_BUFFER,this.mesh.data.verts,gl.STATIC_DRAW)
            
            gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER,this.mesh.buffers.index)
            gl.bufferData(gl.ELEMENT_ARRAY_BUFFER,this.mesh.data.index,gl.STATIC_DRAW)
            
        }
        
        render(){
            
            gl.bindBuffer(gl.ARRAY_BUFFER,this.mesh.buffers.verts)
            gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER,this.mesh.buffers.index)
            
            gl.vertexAttribPointer(positionAttribLocation,3,gl.FLOAT,gl.FALSE,24,0)
            gl.vertexAttribPointer(colorAttribLocation,3,gl.FLOAT,gl.FALSE,24,12)
            
            gl.drawElements(gl.TRIANGLES,this.mesh.indexAmount,gl.UNSIGNED_SHORT,0)
            
        }
    }
    
    let mesh=new Mesh()
    
    mesh.setMeshFromFunction(function(box){
        
        box(0,-5,0,5,1,5,[0,0,0],[0.5, 0.5, 0.5])
        
        box(0,-5,-8,5,1,5,[0,0,0],[0.5, 0.5, 0.5])
        
        box(0,-5,-16,5,1,5,[0,0,0],[0.5, 0.5, 0.5])
        
        box(0,-5,-24,5,1,5,[20,0,0],[0.5, 0.5, 0.5])
         
        box(0,-5,-32,5,1,5,[40,0,0],[0.5, 0.5, 0.5])
         
        box(0,-5,-38,5,1,5,[-40,0,0],[0.5, 0.5, 0.5])
        
        box(0,-5,-46,5,1,5,[-20,0,0],[0.5, 0.5, 0.5])
        
        box(0,-5,-54,5,1,5,[0,0,0],[0.5, 0.5, 0.5])
        
        box(0,-5,-62,5,1,5,[0,0,-50],[0.5, 0.5, 0.5])
        
        box(0,-5,-70,5,1,5,[0,0,50],[0.5, 0.5, 0.5])
        
        box(0,-5,-78,5,1,5,[0,0,-50],[0.5, 0.5, 0.5])
        
        box(0,-5,-86,5,1,5,[0,0,50],[0.5, 0.5, 0.5])
        
        box(0,-5,-94,5,1,5,[20,0,0],[0.5, 0.5, 0.5])
    })
    
    mesh.setBuffers()
    
    const modelViewUniformLocation=gl.getUniformLocation(program,'modelView')
    
    //fov in degrees
    let fov = 70
    
    let projectionMatrix=new Float32Array(m4.perspective(Math.PI*fov/180,width/height,0.1,1000.0))
    
    let time=0
    
    let player={
        
        yaw:0,pitch:0,x:0,y:0,z:0,body:new CANNON.Body({
            
            //dimensions of player
            shape:new CANNON.Box(new CANNON.Vec3(0.5,1,0.5)),
            
            mass:1,
            
            //starting pos of player
            position:new CANNON.Vec3(0,2,0),
            velocity:new CANNON.Vec3(0,0,0),
            fixedRotation:true
            
        }),grounded:true,
    }
    
    player.body.addEventListener('collide',function(e){
        
        if(Math.abs(e.contact.ni.y)>0.5){
            
            player.grounded=true
        }
        
    })
    
    world.addBody(player.body)
    
    uiCanvas.onmousemove=(e)=>{
        if (started){
        //the small numbers are sensitivity
        player.yaw+=e.movementX*0.005
        player.pitch=constrain(player.pitch+e.movementY*0.005,-Math.PI*0.5,Math.PI*0.5)
        }
    }
    
    uiCanvas.onmousedown=()=>{
        
        uiCanvas.requestPointerLock()
    }
    
    let keys={}
    
    document.onkeydown=(e)=>{
        
        keys[e.key.toLowerCase()]=true
    }
    
    document.onkeyup=(e)=>{
        
        keys[e.key.toLowerCase()]=false
    }
    
    let dt,then
    
    function gameLoop(now){
        
        if(!then){
            
            now=window.performance.now()
            then=now
        }
        
        //the little number is the time multiplier
        dt=(now-then)*0.001
        
        //the background color, in the 0-1 range
        gl.clearColor(0,0,0,1)
        
        gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT)
        
        world.step(dt)
        
        //the 70 is movement speed
        let s=dt*speed*(player.grounded?1:0.75),cdir=Math.cos(player.yaw)*s,sdir=Math.sin(player.yaw)*s
        
        //the 0.5 is extra height for the camera to create 'tallness'
        //the true center of the player's body is at player.body.position.y
        player.x=player.body.position.x
        player.y=player.body.position.y+0.5
        player.z=player.body.position.z
        
        if (started){
        
        if(keys.d){
            
            player.body.velocity.x+=cdir
            player.body.velocity.z+=sdir
        }
        
        if(keys.w){
            
            player.body.velocity.x+=sdir
            player.body.velocity.z-=cdir
        }
        
        if(keys.a){
            
            player.body.velocity.x-=cdir
            player.body.velocity.z-=sdir
        }
        
        if(keys.s){
            
            player.body.velocity.x-=sdir
            player.body.velocity.z+=cdir
        }
        
        if(player.grounded){
            
            if(keys[' ']){
                
                player.grounded=false
                //jump power
                player.body.velocity.y = 7
            }
            
            //15 is friction amount on ground
            player.body.velocity.x-=player.body.velocity.x*dt*15
            player.body.velocity.z-=player.body.velocity.z*dt*15
            
        }
        else {
            
            player.body.position.y+=0.001
            
            //friction in air
            player.body.velocity.x-=player.body.velocity.x*dt*7
            player.body.velocity.z-=player.body.velocity.z*dt*7
        }
        
        if (player.y < -25){
            main();
            gameLoop();
        }
        
        timeSpeed += 1;
        }
        
        timeTextEl.innerHTML = "Time: "+timeSpeed;
        versionEl.innerHTML = "Version 1.0"
        
        function start(){
            started = true;
            how_toEl.style.top = "-400px"
            headEl.style.top = "-400px"
        }
        
        if (!started){
            how_toEl.style.top = "200px"
            how_toEl.style.left = "190px"
            how_toEl.innerHTML = "<h2>How To</h2><p>Use wasd to move, mouse to look around. Avoid red lava, and try to get to the green dot as fast as you can. Click to start.</p>"
            document.addEventListener("click", start)
            headEl.style.top = "60px"
            headEl.style.left = "28%"
            headEl.innerHTML = "<h1>Parkour 3D</h1>"
        }
        
        /*
        if (cubeCollide(player.x, player.y, player.z, 0.5, 1, 0.5, 0, -4, -26, 1, 1, 1)){
            
        }*/
        
        viewMatrix=m4.xRotate(m4.yRotate(m4.identity(),player.yaw),player.pitch)
        
        viewMatrix=m4.translate(viewMatrix,-player.x,-player.y,-player.z)
        
        viewMatrix=m4.mult(projectionMatrix,viewMatrix)
        
        gl.uniformMatrix4fv(modelViewUniformLocation,gl.FALSE,viewMatrix)
    
        mesh.render()
        
        then=now
        ctx.drawImage(gl.canvas,0,0)
        
        window.parent.raf=window.requestAnimationFrame(gameLoop)
    }
    
    if(window.parent.raf){
        
        window.cancelAnimationFrame(window.parent.raf)
    }
    
    gameLoop()
}

main()

</script>

<!--For restart button-->
<script></script>
    </body>
</html>
