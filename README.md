# php-theme


Hello everyone , this is my custom theme for wordpress. Its contains blog, shop , basket , allow to registrate new users and set profile information. If you 
want to try this theme download archive clock-theme to your folder with themes , and run sql statement from databaseSQL.sql in phpMyAdmin. 
Theme overview:



1) In this theme you can like or dislike blog posts via ajax requests:

HTML themplate for likes and dislikes
```
<ul class="likes">
	<li class="likes__item likes__item--like">
		<button id="like-button" class="btn btn-info" data-id="<?php echo get_the_ID();?>" data-action="like">
			Like: 
			<span class="counter" id="like-button-counter" >
				<?php 
					global $likes;
					echo $likes; 
				?>
			</span>
		</button>
	</li>
	<li class="likes__item likes__item--dislike">
		<button id="dislike-button" class="btn btn-info"   data-action="dislike">
			Dislike: 
			<span class="counter" id="dislike-button-counter" >
			<?php 
					global $dislikes;
					echo $dislikes; 
				?>
			</span>
        </button>
	</li>
</ul>
```

To allow likes and dislikes I add function 'myajax_data' to send  user_id and url  to my frontend:
```
add_action( 'wp_enqueue_scripts', 'myajax_data', 99 );
function myajax_data(){

	wp_localize_script('jquery', 'myajax', 
		array(
      'url' => admin_url('admin-ajax.php'),
      'user_id'=>get_current_user_id(),
      'nonce' => wp_create_nonce('myajax-nonce'),
		)
	);  

}

```


Then I implement jquery function thats run when like or dislike button clicked:
```
add_action('wp_footer', 'my_action_javascript'); 
function my_action_javascript() {
  ?>
  <script>
	jQuery(document).ready(function($) {
    var id  = $( "#like-button" ).data("id"); 
    

		var data = {
		  action  : 'my_action',
     user_id : myajax.user_id,
     post_id : id,
     nonce_code : myajax.nonce,
		};

    $( "#like-button , #dislike-button" ).click(function(event) {
      var action_user = $(event.target).data('action');
      if (typeof action_user === 'undefined' ){
        alert('Please, take a few moments , page not loaded all ')
      }
      if (data.user_id !== "0" && (action_user === "like" || action_user === "dislike")){
        var action_user = $(event.target).data('action');
            data.action_user = action_user;
           // console.log(data);

          jQuery.post( myajax.url, data, function(response) {
            $( 'button[data-action='+action_user+']>span' ).text(response);
            });
      } else {
         alert("Please , log in first ");
      }


        });
	});
	</script>
	<?php
}

```
 Once it was clicked function sends ajax request with data. The responce of request will be paste in respective tag
 To catch this request I use 'my_action_callback' function .  'my_action_callback' function parse variables from $_POST superglobal and make safe
 query to database via $wpdb wordpress API.

```
add_action('wp_ajax_my_action', 'my_action_callback');

function my_action_callback() {
  global $wpdb;
  if( ! wp_verify_nonce( $_POST['nonce_code'], 'myajax-nonce' ) ) die( 'Stop!');
  
 $user_id = intval( $_POST['user_id'] );
 $post_id = intval($_POST['post_id']) ;
 $action_user = $_POST['action_user'] ;



  $prepare_query  = $wpdb->prepare("INSERT INTO `wp_likes_dislikes` (`post_id`, `user_id`, `action`) VALUES ('%d', '%d', '%s')",$post_id,$user_id,$action_user);

  $query = $wpdb->query($prepare_query);


	$query = $wpdb->get_var("SELECT COUNT( * ) FROM `wp_likes_dislikes` WHERE `action`='{$action_user}' AND `post_id`={$post_id} ");

  
  echo $query;

  wp_die();
}

```

2) Telegram bot for send chekouts :


In this piece of code I parse variables from $_POST superglobal and make text message for telegram bot.Than I sand message via fopen function. If fopen return true I clean 
basket and sending header to redirect user to another page. In my first tries I set parse_mode of bot equal to HTML,whitch cause erorr 400  Bad request. To find out what is wrong I 
used cURL extention for php. This extention showed me that there is a parse erorr, when I change my text message and parse_mode. Than I fixed bugs and it start working.

```
if (isset($_POST['submit'])){
  

	$product_name = $_POST['product-name'];
	$product_quantity = $_POST['product-quantity'];
	$product_price = $_POST['product-price'];
	$user_name = $_POST['user_name'];
	$token = "**************************";
	$chat_id = "***************"; 
	$user_phone_number = $_POST['phone-number'];
	


// print_r($product_name) ;
// echo "</br>";
// print_r($product_quantity);
// echo "</br>";
// print_r($product_price);

$length = count($product_name);

$text = "";

for ($i = 0; $i < $length; $i++) {
    $text =$text."*".$product_name[$i]."*".",    ".$product_quantity[$i]." units with price ".$product_price[$i].";  ";
}

 $text=$text."*  Total sum:*".array_sum($product_price);
 $text=$text."*  User phone number: *".$user_phone_number;
 $text=$text."*  User name: *".$user_name;

//  $ch = curl_init();
//  curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
//  curl_setopt($ch, CURLOPT_URL, 
//  "https://api.telegram.org/bot{$token}/sendMessage?chat_id={$chat_id}&parse_mode=markdown&text={$text}"
//  );
//  $content = curl_exec($ch);
//  echo $text;
//  print_r($content);

// "https://api.telegram.org/bot{$token}/sendMessage?chat_id={$chat_id}&parse_mode=html&text={$text}";


$send_to_telegram = fopen("https://api.telegram.org/bot{$token}/sendMessage?chat_id={$chat_id}&parse_mode=markdown&text={$text}","r");

//echo "https://api.telegram.org/bot{$token}/sendMessage?chat_id={$chat_id}&parse_mode=html&text={$text}";


if($send_to_telegram){
	WC()->cart->empty_cart();
	header("location:http://rogaldorn-shop.com/chekout-success");
	exit;
}


}

```

3) SMTP server for password reset :
For sending emails I use gmail SMTP server:
```
add_action( 'phpmailer_init', 'my_phpmailer_example' );
function my_phpmailer_example( $phpmailer ) {

	$phpmailer->isSMTP();     
	$phpmailer->Host = 'smtp.gmail.com';
	$phpmailer->SMTPAuth = true; // Force it to use Username and Password to authenticate
	$phpmailer->Port = 587;
  $phpmailer->Username = 'your gmail email';                 // SMTP username
  $phpmailer->Password = 'your gmail password';                           // SMTP password
  $phpmailer->SMTPSecure = 'tls';
  $phpmailer->isHTML(true);                                  // Set email format to HTML

  $phpmailer->SMTPOptions = array(
    'ssl' => array(
        'verify_peer' => false,
        'verify_peer_name' => false,
        'allow_self_signed' => true
    )
);

	// Additional settings…
	//$phpmailer->SMTPSecure = "tls"; // Choose SSL or TLS, if necessary for your server
	//$phpmailer->From = "you@yourdomail.com";
	//$phpmailer->FromName = "Your Name";
}

```
Page :

```
<?php 

$user_login = $_SERVER["QUERY_STRING"];
$user_login = explode("=", $user_login);
$user_login =str_replace('/', '', $user_login[1]);

if (isset($_POST['submit-email'])){
    global $wpdb;
    $email = $_POST['email'];
    $subject = "Password reset";
    $rand = rand(100000,500000);
    $message = "Here is your code for password reset ".$rand;

    $wpdb->insert(
        'tmp-password',
        array( 'email' => $email, 'code' => $rand ),
        array( '%s', '%d' )
    );

    wp_mail($email, $subject, $message, $headers, $attachments);

    ?> 
    <form class="p-2 container" name="loginform" id="loginform" action="<?php the_permalink(); ?>" method="POST">
        <div class="form-group row flex-column col-md-3">
            <label for="exampleInputPassword1">We send code to your email <?php echo $email; ?>  for password reset </label>
            <input name="secret-code"  type="text" require class="form-control " placeholder="Secret code" >
            <input type="hidden" name="email" value="<?php echo $email ?>" >
        </div>
        <button name="submit-code" type="submit" class="btn btn-primary">Submit</button>
    </form>
    <?php
} elseif (isset($_POST['submit-code'])){
    global $wpdb;
    $secret_code = $_POST['secret-code'];
    $email_from_request = $_POST['email'];
    $table = 'tmp-password';
	$user_email_from_table = $wpdb->get_var($wpdb->prepare("SELECT `email` FROM `tmp-password` WHERE `code`=%d ",$secret_code));
    //Delete row from table
    
    //reset field
    if ($user_email_from_table === $email_from_request){
        ?> 
        <form class="p-2 container" name="loginform" id="loginform" action="<?php the_permalink(); ?>" method="POST">
            <div class="form-group row flex-column col-md-3">
                <label for="exampleInputPassword1">Type new password : </label>
                <input name="password"  type="password" require class="form-control " placeholder="password" >
                <input type="hidden" name="email" value="<?php echo $email_from_request ?>" >
            </div>
            <button name="submit-new-password" type="submit" class="btn btn-primary">Submit</button>
        </form>
        <?php
    } else {
        echo "<script>alert('Bad password')</script>";
        ?> 
        <form class="p-2 container" name="loginform" id="loginform" action="<?php the_permalink(); ?>" method="POST">
            <div class="form-group row flex-column col-md-3">
                <label for="exampleInputPassword1">We send code to your email <?php echo $email_from_request; ?>  for password reset </label>
                <input name="secret-code"  type="text" require class="form-control " placeholder="Secret code" >
                <input type="hidden" name="email" value="<?php echo $email_from_request ?>" >
            </div>
            <button name="submit-code" type="submit" class="btn btn-primary">Submit</button>
        </form>
        <?php
    }


} elseif (isset($_POST['submit-new-password'])) {
    $user_id = get_user_by('email', $_POST['email'])->ID;
    wp_set_password( $_POST['password'], $user_id );
    ?> 
        <h3>Password was updated !</h3>
    <?php

} else {
    ?>
        <form class="p-2 container" name="loginform" id="loginform" action="<?php the_permalink(); ?>" method="POST">
            <div class="form-group row flex-column col-md-3">
            <label for="exampleInputPassword1">Your email <?php echo $user_login;  ?></label>
            <input name="email" type="email" require class="form-control " id="exampleInputPassword1" placeholder="email">

            </div>
            <button name="submit-email" type="submit" class="btn btn-primary">Send reset email</button>
        </form>
    <?php
}?>

```






