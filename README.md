package com.example.windigo;

import net.minecraft.world.entity.Mob;
import net.minecraft.world.entity.MobType;
import net.minecraft.world.entity.ai.attributes.Attributes;
import net.minecraft.world.entity.ai.goal.Goal;
import net.minecraft.world.entity.ai.goal.RandomStrollGoal;
import net.minecraft.world.entity.ai.goal.target.NearestAttackableTargetGoal;
import net.minecraft.world.entity.player.Player;
import net.minecraft.world.effect.MobEffects;
import net.minecraft.world.effect.MobEffectInstance;
import net.minecraft.world.damagesource.DamageSource;
import net.minecraft.sounds.SoundEvent;
import net.minecraft.world.level.Level;
import net.minecraft.world.entity.EntityType;
import net.minecraft.world.phys.AABB;

import javax.annotation.Nullable;
import java.util.EnumSet;
import java.util.List;
import java.util.Random;

/**
 * Minimal illustrative Windigo entity.
 *
 * Notes:
 * - This is a conceptual example for a Forge-style mod.
 * - You still must register the EntityType, model, renderer, sounds, and client resources.
 * - Replace TODOs with your registration and resource names.
 */
public class WindigoEntity extends Mob {

    private int howlCooldown = 0;

    protected WindigoEntity(EntityType<? extends Mob> type, Level world) {
        super(type, world);
        this.setPersistenceRequired(); // avoid despawn
    }

    // Set base attributes. Register these on entity creation/registration too.
    public static void createAttributes(net.minecraft.world.entity.ai.attributes.AttributeSupplier.Builder builder) {
        builder
            .add(Attributes.MAX_HEALTH, 30.0D)
            .add(Attributes.MOVEMENT_SPEED, 0.25D) // walking
            .add(Attributes.ATTACK_DAMAGE, 6.0D)
            .add(Attributes.FOLLOW_RANGE, 24.0D);
    }

    @Override
    protected void registerGoals() {
        // Random wandering / stalking
        this.goalSelector.addGoal(1, new WindigoStalkGoal(this, 1.0D));
        this.goalSelector.addGoal(2, new RandomStrollGoal(this, 0.8D));
        // Target nearest player when conditions met
        this.targetSelector.addGoal(1, new NearestAttackableTargetGoal<>(this, Player.class, true));
    }

    @Override
    public void tick() {
        super.tick();

        if (!this.level.isClientSide) {
            // Howl periodically at night or when a player is near
            if (howlCooldown > 0) howlCooldown--;
            List<Player> nearby = this.level.getEntitiesOfClass(Player.class, new AABB(this.blockPosition()).inflate(16));
            boolean isNight = this.level.isNight();

            if ((isNight || !nearby.isEmpty()) && howlCooldown == 0) {
                this.level.playSound(null, this.blockPosition(), getHowlSound(), net.minecraft.sounds.SoundSource.HOSTILE, 1.0F, 1.0F);
                howlCooldown = 20 * (40 + this.random.nextInt(40)); // 40-80s cooldown
            }

            // Special teleport or vanish when low health (adds mystery)
            if (this.getHealth() < this.getMaxHealth() * 0.2 && this.random.nextFloat() < 0.02F) {
                teleportAway();
            }
        }
    }

    private void teleportAway() {
        // Simple teleport logic - tries a few random nearby spots
        Random rnd = this.random;
        for (int i = 0; i < 10; i++) {
            double dx = this.getX() + (rnd.nextDouble() - 0.5D) * 16.0D;
            double dy = Math.max(1.0D, this.getY() + (rnd.nextDouble() - 0.5D) * 8.0D);
            double dz = this.getZ() + (rnd.nextDouble() - 0.5D) * 16.0D;
            if (this.randomTeleport(dx, dy, dz, true)) {
                // play a spooky sound or particle - left as TODO
                return;
            }
        }
    }

    @Override
    public boolean doHurtTarget(net.minecraft.world.entity.Entity target) {
        boolean result = super.doHurtTarget(target);
        if (result && target instanceof Player) {
            // Apply a "sanity" or weakness effect to the player on hit
            Player p = (Player) target;
            p.addEffect(new MobEffectInstance(MobEffects.MOVEMENT_SLOWDOWN, 20 * 6, 1)); // slowness 6 sec
            // TODO: hook into a custom sanity system if you have one
        }
        return result;
    }

    @Nullable
    protected SoundEvent getHowlSound() {
        // TODO: return your registered SoundEvent
        return null;
    }

    // Example of an inner Goal that makes the Windigo stalk slowly until close, then sprint
    static class WindigoStalkGoal extends Goal {
        private final WindigoEntity windigo;
        private final double speedModifier;
        private Player target;

        WindigoStalkGoal(WindigoEntity w, double speed) {
            this.windigo = w;
            this.speedModifier = speed;
            this.setFlags(EnumSet.of(Flag.MOVE, Flag.LOOK));
        }

        @Override
        public boolean canUse() {
            this.target = this.windigo.level.getNearestPlayer(this.windigo, 24.0D);
            if (this.target == null) return false;
            // Prefer to hunt at night or if target is alone / low sanity
            boolean isNight = this.windigo.level.isNight();
            return isNight || this.windigo.random.nextFloat() < 0.5F;
        }

        @Override
        public void tick() {
            if (this.target == null) return;
            double dist = this.windigo.distanceToSqr(this.target);
            // If far: move slowly (stalk). If close: sprint toward player.
            if (dist > 9.0D) {
                this.windigo.getNavigation().moveTo(this.target, this.speedModifier * 0.6D);
            } else {
                this.windigo.getNavigation().moveTo(this.target, this.speedModifier * 1.4D);
            }

            // Occasionally apply a visual "fear" debuff to the player to simulate sanity loss
            if (this.windigo.random.nextInt(200) == 0) {
                this.target.addEffect(new MobEffectInstance(MobEffects.BLINDNESS, 40, 0));
            }
        }
    }

    @Override
    public boolean isInColdBiome() {
        // Pseudocode: prefer spawning in cold/snow biomes; implement via spawn rules
        return true; // leave spawn filtering to registration code
    }

    @Override
    public MobType getMobType() {
        return MobType.UNDEFINED;
    }

    @Override
    public boolean hurt(DamageSource source, float amount) {
        // React strangely to fire or silver items if you want special vulnerabilities
        return super.hurt(source, amount);
    }
}
